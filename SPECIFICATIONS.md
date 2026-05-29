# Project Specification Document
## Standalone Beehive Monitoring System (Local-Only / No Cloud Dependency)

**Document type:** Functional + Technical Specification  
**Prepared by:** Project Management / Solution Architecture  
**Version:** 1.0  
**Date:** 2026-05-11  
**Project status:** Implementation-ready specification

---

# Table of Contents

1. [Document Purpose](#1-document-purpose)
2. [Project Overview](#2-project-overview)
3. [Scope](#3-scope)
4. [Stakeholders and Users](#4-stakeholders-and-users)
5. [System Context and Architecture](#5-system-context-and-architecture)
6. [Functional Requirements](#6-functional-requirements)
7. [Non-Functional Requirements](#7-non-functional-requirements)
8. [Hardware Specification](#8-hardware-specification)
9. [Software Specification](#9-software-specification)
10. [Communication Specification](#10-communication-specification)
11. [Data Storage and InfluxDB Data Model](#11-data-storage-and-influxdb-data-model)
12. [Security Specification](#12-security-specification)
13. [Reliability and Fault-Tolerance Specification](#13-reliability-and-fault-tolerance-specification)
14. [Power Management Specification](#14-power-management-specification)
15. [Implementation Plan](#15-implementation-plan)
16. [Acceptance Criteria](#16-acceptance-criteria)
17. [Acceptance Test Plan](#17-acceptance-test-plan)
18. [Risks and Mitigations](#18-risks-and-mitigations)
19. [Operations and Maintenance](#19-operations-and-maintenance)
20. [Deliverables](#20-deliverables)
21. [Recommended Initial Bill of Materials](#21-recommended-initial-bill-of-materials-indicative)
22. [Appendix A — Example Node Wake Cycle](#22-appendix-a--example-node-wake-cycle)
23. [Appendix B — Example Requirement Traceability Snapshot](#23-appendix-b--example-requirement-traceability-snapshot)
24. [Final Implementation Recommendation](#24-final-implementation-recommendation)
25. [Definition of Done](#25-definition-of-done)

---

# 1. Document Purpose

This document defines the **project specifications** for a **standalone beehive monitoring system** designed for **local operation without cloud dependency**. It is intended to serve as the primary reference for the implementation team.

It covers:

- Project overview and objectives
- Functional requirements
- Non-functional requirements
- Hardware requirements
- Software requirements
- Data architecture
- Security and reliability requirements
- Implementation steps
- Testing and acceptance criteria

The goal is to ensure that the implementing engineer or team can:

- Understand the system quickly
- Build it consistently
- Extend it modularly
- Validate it against clear acceptance criteria

---

# 2. Project Overview

## 2.1 Project Summary

The system consists of **sensor nodes installed in beehives**, based on **ESP32 or ESP8266 microcontrollers**, which collect environmental and operational data such as:

- Temperature
- Humidity
- Weight
- Sound or vibration
- Battery voltage

Each node transmits data via **MQTT** over a local wireless network to a **Raspberry Pi gateway**, which hosts:

- **Mosquitto** MQTT broker
- **InfluxDB** time-series database
- Optional **Grafana** dashboards
- Optional **Node-RED** or Python ingestion service

The system is designed to:

- Operate **without internet or cloud services**
- Run for long periods **unattended**
- Consume **minimal power**
- Be easy to extend with new sensors and nodes

## 2.2 Business / Operational Objectives

The project must provide:

- Continuous local monitoring of hive conditions
- Historical time-series data storage
- Low-maintenance field operation
- Modular architecture for future growth
- Reliable data capture even when connectivity is temporarily unavailable

## 2.3 High-Level Goals

- Build a **local-first** IoT monitoring platform
- Support **multiple hives**
- Ensure **long battery life**
- Allow **troubleshooting and maintenance**
- Keep the implementation **simple, robust, and extensible**

---

# 3. Scope

## 3.1 In Scope

The following are **in scope** for this project:

- ESP-based beehive sensor nodes
- Local Wi-Fi-based MQTT communication
- Raspberry Pi-based MQTT broker
- Local InfluxDB storage
- Sensor node firmware
- Gateway software stack
- Data model design
- Local dashboards
- Offline tolerance and buffering
- Local security hardening
- Acceptance testing

## 3.2 Out of Scope

The following are **out of scope** unless explicitly added later:

- Cloud integration
- Remote internet access
- Mobile apps
- OTA updates over the internet
- AI-based anomaly detection
- Automated hive control (heating, ventilation, feeding, etc.)
- Raw long-duration audio archiving
- Cellular or LoRaWAN connectivity

---

# 4. Stakeholders and Users

## 4.1 Stakeholders

- **Project owner / beekeeper**
- **System implementer / embedded engineer**
- **System maintainer / operator**
- **Data user / analyst**

## 4.2 Primary Users

- Beekeeper or operator monitoring hive condition
- Technical maintainer troubleshooting system status
- Analyst reviewing trends from stored data

---

# 5. System Context and Architecture

## 5.1 Textual Architecture Diagram

```text
┌────────────────────────────────────────────────────────────────┐
│                          BEEHIVES                              │
│                                                                │
│  ┌──────────────────────┐      ┌──────────────────────┐        │
│  │ Hive Node A          │      │ Hive Node B          │        │
│  │ ESP32 / ESP8266      │      │ ESP32 / ESP8266      │        │
│  │----------------------│      │----------------------│        │
│  │ Temp / Humidity      │      │ Temp / Humidity      │        │
│  │ Weight               │      │ Weight               │        │
│  │ Sound / Vibration    │      │ Sound / Vibration    │        │
│  │ Battery Voltage      │      │ Battery Voltage      │        │
│  └──────────┬───────────┘      └──────────┬───────────┘        │
│             │ Wi-Fi / MQTT                │ Wi-Fi / MQTT       │
└─────────────┼─────────────────────────────┼────────────────────┘
              │                             │
              ▼                             ▼
┌────────────────────────────────────────────────────────────────┐
│                    LOCAL NETWORK / IoT WLAN                    │
│                                                                │
│  Dedicated Wi-Fi AP / isolated LAN segment recommended         │
└───────────────────────────────┬────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────┐
│                      RASPBERRY PI GATEWAY                      │
│                                                                │
│  ┌───────────────────────┐   ┌──────────────────────────────┐  │
│  │ Mosquitto MQTT Broker │-->│ Ingestion Service            │  │
│  │                       │   │ (Telegraf / Node-RED/Python) │  │
│  └───────────────────────┘   └──────────────┬───────────────┘  │
│                                             │                  │
│                                             ▼                  │
│                                  ┌──────────────────────────┐  │
│                                  │ InfluxDB                 │  │
│                                  │ Time-series storage      │  │
│                                  └──────────────┬───────────┘  │
│                                                 │              │
│                                                 ▼              │
│                                  ┌──────────────────────────┐  │
│                                  │ Grafana (optional)       │  │
│                                  │ Local dashboards         │  │
│                                  └──────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

## 5.2 Data Flow Diagram

```text
[ESP Node wakes]
      ↓
[Power sensors]
      ↓
[Sample sensors]
      ↓
[Validate readings]
      ↓
[Connect to Wi-Fi]
      ↓
[Connect to MQTT]
      ↓
[Publish telemetry + status]
      ↓
[Gateway receives MQTT]
      ↓
[Ingestion service parses payload]
      ↓
[InfluxDB stores time-series data]
      ↓
[Grafana displays local dashboards]
      ↓
[ESP node enters deep sleep]
```

## 5.3 Component Roles

### Sensor Node
Responsible for:

- Sampling sensors
- Packaging telemetry
- Power management
- Buffering when offline
- MQTT publishing

### Raspberry Pi
Responsible for:

- Running MQTT broker
- Storing time-series data
- Running dashboards
- Centralized local monitoring
- Local security controls

---

# 6. Functional Requirements

Each requirement is identified so it can be traced to implementation and testing.

## 6.1 Sensor Node Functional Requirements

### FR-001 — Temperature Measurement
The system shall measure hive temperature.

- Measurement unit: °C
- Sampling interval configurable
- Sensor reading included in telemetry when sensor is enabled

### FR-002 — Humidity Measurement
The system shall measure hive humidity.

- Measurement unit: %
- Sensor reading included in telemetry when enabled

### FR-003 — Weight Measurement
The system shall support weight measurement using a scale interface.

- Measurement unit: kg
- Calibration support required
- Raw and calibrated values should be available

### FR-004 — Sound / Vibration Monitoring
The system shall support sound or vibration activity monitoring.

- The node may calculate summary metrics locally
- Raw continuous audio streaming is not required

### FR-005 — Battery Voltage Monitoring
The system shall measure and report node battery voltage.

- Measurement unit: V
- Used for monitoring and adaptive power behavior

### FR-006 — Telemetry Publishing
The node shall publish sensor data over MQTT to the local broker.

### FR-007 — Status Reporting
The node shall publish device health/status information periodically.

### FR-008 — Availability Reporting
The node shall report its online/offline state using MQTT retained availability messages and last-will functionality.

### FR-009 — Local Configuration Support
The node shall support receiving configuration values through MQTT or local stored configuration.

### FR-010 — Deep Sleep Operation
The node shall enter deep sleep between measurement cycles to reduce power consumption.

### FR-011 — Sensor Enable / Disable Modularity
The firmware shall support enabling/disabling sensors per node without codebase redesign.

### FR-012 — Offline Buffering
The node shall support storing unsent telemetry locally if MQTT delivery fails.

### FR-013 — Sequence Tracking
The node shall include a sequence counter in published telemetry to help detect missing messages.

## 6.2 Gateway Functional Requirements

### FR-014 — Local MQTT Broker
The Raspberry Pi shall run a local MQTT broker.

### FR-015 — Local Data Storage
The gateway shall store telemetry in InfluxDB.

### FR-016 — Ingestion Service
The gateway shall run a service to consume MQTT messages and write structured time-series points to InfluxDB.

### FR-017 — Local Dashboards
The system shall support local dashboards for visualizing sensor and health data.

### FR-018 — Local Administration
The system shall support local-only administration and troubleshooting.

### FR-019 — Multi-Hive Support
The gateway shall support receiving and storing data from multiple hives simultaneously.

## 6.3 Configuration and Management Functional Requirements

### FR-020 — Per-Node Identification
Every node shall have a unique device ID and be associated with a unique hive ID.

### FR-021 — Configurable Sampling Intervals
Sampling intervals shall be configurable per node.

### FR-022 — Error Reporting
Sensor and communication errors shall be reported in node status or telemetry health fields.

### FR-023 — Restart Recovery
After a node or gateway reboot, the system shall resume normal operation automatically.

---

# 7. Non-Functional Requirements

## 7.1 Power Requirements

### NFR-001 — Low Power Operation
The sensor node shall be optimized for low power consumption using deep sleep and minimal radio activity.

### NFR-002 — Battery-Friendly Operation
The node shall avoid continuous high-power activities such as permanent Wi-Fi connectivity or continuous audio streaming.

## 7.2 Reliability Requirements

### NFR-003 — Unattended Operation
The system shall be designed for long-term unattended operation.

### NFR-004 — Temporary Connectivity Loss Tolerance
The system shall tolerate temporary Wi-Fi, MQTT, or gateway outages without catastrophic data loss.

### NFR-005 — Automatic Recovery
Nodes and services shall recover automatically after temporary faults or restarts.

## 7.3 Maintainability Requirements

### NFR-006 — Modularity
The solution shall be modular so that new sensors can be added with minimal redesign.

### NFR-007 — Serviceability
The solution shall expose enough status and logging information to allow diagnosis of failures.

### NFR-008 — Traceability
Telemetry shall include sufficient metadata to identify source device, hive, firmware version, and measurement context.

## 7.4 Performance Requirements

### NFR-009 — Efficient Payloads
MQTT payloads shall be compact enough to minimize airtime and energy consumption.

### NFR-010 — Acceptable Latency
Telemetry shall normally appear in the database within 10 seconds of successful MQTT publication under nominal local network conditions.

## 7.5 Security Requirements

### NFR-011 — Local Authentication
MQTT access shall require authentication.

### NFR-012 — Network Isolation
The system shall be deployable on an isolated local network or VLAN.

### NFR-013 — No Internet Dependency
Core operation shall not depend on internet connectivity.

---

# 8. Hardware Specification

## 8.1 Sensor Node Hardware Requirements

### 8.1.1 Recommended Microcontroller

#### Preferred
- **ESP32**
  - Recommended for new designs
  - Better memory and peripheral support
  - Better support for more complex sensors
  - Suitable for deep sleep

#### Allowed
- **ESP8266**
  - Acceptable for simpler nodes
  - Not preferred for multi-sensor deployments

**Requirement:** Final design shall support at least one ESP32-based node.

### 8.1.2 Recommended Sensors

#### Temperature / Humidity
Preferred sensors:

- SHT31
- SHT35
- BME280

**Requirement:** Sensor must provide stable, repeatable digital output.

#### Weight
Recommended:

- Load cell(s) with HX711 amplifier

**Requirement:** Weight subsystem must support calibration and raw value access.

#### Sound / Vibration
Acceptable options:

- I²S MEMS microphone
- Piezo vibration sensor
- Accelerometer (e.g. LIS3DH, ADXL345)

**Requirement:** System should compute summary features locally rather than transmit raw continuous data.

#### Battery Voltage
Recommended:

- Resistor divider to ADC
- Preferably switched or gated to reduce drain

**Requirement:** Battery voltage monitoring shall be available on all battery-powered nodes.

### 8.1.3 Power Subsystem

Recommended options:

- LiFePO₄ battery
- 18650 Li-ion battery with protection
- Optional solar charging subsystem

**Requirement:** Power subsystem shall be suitable for unattended outdoor operation.

### 8.1.4 Enclosure Requirements

- Outdoor-capable enclosure, preferably IP65 or better
- Cable strain relief and glands
- Corrosion-resistant connectors
- Mechanical protection for scale and external sensors
- Moisture management (desiccant / ventilation strategy as appropriate)

### 8.1.5 Node Hardware Illustration

```text
┌───────────────────────────────────────────┐
│ Sensor Node Enclosure                     │
│                                           │
│  ESP32 Module                             │
│  ├── I²C → Temp/Humidity Sensor           │
│  ├── ADC → Battery Divider                │
│  ├── GPIO → HX711                         │
│  ├── I²S/GPIO → Audio/Vibration Sensor    │
│  └── GPIO → Sensor Power MOSFET           │
│                                           │
│  Battery / Solar Input                    │
│  External Sensor Connectors               │
└───────────────────────────────────────────┘
```

## 8.2 Raspberry Pi Hardware Requirements

Recommended:

- Raspberry Pi 4 (2 GB or 4 GB minimum recommended)
- Raspberry Pi 5 acceptable
- External SSD strongly recommended for database durability
- Stable power supply
- Ethernet preferred when available

### Minimum Gateway Capability
The gateway hardware must support concurrent operation of:

- Mosquitto
- InfluxDB
- Ingestion service
- Optional Grafana

---

# 9. Software Specification

## 9.1 Sensor Node Firmware Requirements

The ESP firmware shall provide:

- Wi-Fi connectivity
- MQTT client connectivity
- Sensor initialization and sampling
- Telemetry packaging
- Configurable sampling intervals
- Deep sleep
- Error handling
- Offline buffering
- Watchdog-safe operation
- Modular sensor architecture

## 9.2 Firmware Module Structure (Recommended)

```text
Firmware
├── ConfigManager
├── SensorManager
│   ├── TempHumiditySensor
│   ├── WeightSensor
│   ├── BatterySensor
│   ├── SoundSensor
│   └── VibrationSensor
├── PowerManager
├── WiFiManager
├── MQTTManager
├── BufferManager
└── HealthManager
```

## 9.3 Raspberry Pi Software Stack

Required services:

- **Mosquitto** — local MQTT broker
- **InfluxDB** — time-series database
- **Ingestion service** — Telegraf, Node-RED, or Python service
- **Grafana** (recommended) — local dashboards

Optional services:

- systemd service wrappers
- local backup scripts
- log rotation
- health monitoring scripts

---

# 10. Communication Specification

## 10.1 MQTT Topic Structure

The system shall use a structured MQTT namespace:

```text
beehive/{site_id}/{apiary_id}/{hive_id}/telemetry
beehive/{site_id}/{apiary_id}/{hive_id}/status
beehive/{site_id}/{apiary_id}/{hive_id}/availability
beehive/{site_id}/{apiary_id}/{hive_id}/config
beehive/{site_id}/{apiary_id}/{hive_id}/command
```

### Example

```text
beehive/site01/apiary01/hive-001/telemetry
beehive/site01/apiary01/hive-001/status
beehive/site01/apiary01/hive-001/availability
```

## 10.2 MQTT QoS Policy

Recommended defaults:

- Telemetry: QoS 1
- Status: QoS 1
- Availability: QoS 1 retained
- Config: QoS 1 retained
- Commands: QoS 1

If ultra-low-power optimization is required, telemetry may be reduced to QoS 0 after validation.

## 10.3 Availability Messaging

Each node shall use:

- MQTT Last Will and Testament (LWT)
- Retained availability state

Payloads:

```text
online
offline
```

## 10.4 Example Telemetry Payload

```json
{
  "device_id": "hive-001-node-a",
  "site_id": "site01",
  "apiary_id": "apiary01",
  "hive_id": "hive-001",
  "timestamp": "2026-05-11T16:30:00Z",
  "sequence": 18422,
  "sensors": {
    "temperature_c": 34.7,
    "humidity_percent": 58.2,
    "weight_kg": 42.85,
    "battery_v": 3.91,
    "sound_rms": 0.034,
    "sound_peak_frequency_hz": 245,
    "vibration_index": 12.4
  },
  "health": {
    "rssi_dbm": -67,
    "uptime_s": 8,
    "free_heap_bytes": 153240,
    "reset_reason": "deep_sleep",
    "firmware_version": "1.2.0",
    "sensor_errors": []
  }
}
```

## 10.5 Example Status Payload

```json
{
  "device_id": "hive-001-node-a",
  "firmware_version": "1.2.0",
  "battery_v": 3.91,
  "rssi_dbm": -67,
  "last_sample_ok": true,
  "sensor_errors": [],
  "boot_count": 18422
}
```

## 10.6 Command Payload Examples

```json
{
  "command": "reboot"
}
```

```json
{
  "command": "set_interval",
  "sample_interval_seconds": 900
}
```

```json
{
  "command": "tare_scale"
}
```

---

# 11. Data Storage and InfluxDB Data Model

## 11.1 Data Modelling Principles

- Use **measurements** for logical data categories
- Use **tags** for identifiers and filtering
- Use **fields** for changing numeric values
- Avoid high-cardinality tags
- Keep schema consistent across all nodes

## 11.2 Recommended Measurements

### Measurement 1: `hive_environment`

**Tags:**
- site_id
- apiary_id
- hive_id
- device_id

**Fields:**
- temperature_c
- humidity_percent
- pressure_hpa (optional)

### Measurement 2: `hive_weight`

**Tags:**
- site_id
- apiary_id
- hive_id
- device_id
- scale_id

**Fields:**
- weight_kg
- raw_adc
- sample_count

### Measurement 3: `hive_activity`

**Tags:**
- site_id
- apiary_id
- hive_id
- device_id
- sensor_type

**Fields:**
- sound_rms
- sound_peak_frequency_hz
- vibration_index
- activity_score

### Measurement 4: `node_health`

**Tags:**
- site_id
- apiary_id
- hive_id
- device_id
- firmware_version

**Fields:**
- battery_v
- battery_percent
- rssi_dbm
- uptime_s
- free_heap_bytes
- boot_count
- publish_duration_ms
- sensor_error_count

## 11.3 Tag and Field Rules

### Use tags for:
- Hive identity
- Site identity
- Device identity
- Firmware version (optional but useful for filtering)

### Use fields for:
- Numeric sensor readings
- Battery voltage
- RSSI
- Error counts
- Performance metrics

### Do not use tags for:
- Sequence number
- Timestamp
- Dynamic error messages
- Raw unbounded identifiers

## 11.4 Retention Strategy

Recommended:

- **Raw data bucket**: 30 to 90 days
- **Downsampled bucket**: 1 to 2 years or more
- Optional daily aggregates for long-term trends

---

# 12. Security Specification

## 12.1 Network Security

The deployment shall support:

- Dedicated IoT SSID or VLAN
- No internet exposure
- No port forwarding
- Firewall rules on Raspberry Pi
- Limited access from sensor nodes to gateway services only

## 12.2 MQTT Security

Required:

- Username/password authentication
- Per-node credentials
- ACL-restricted topics
- No anonymous publish access in production

### ACL Example (conceptual)

- A node for `hive-001` may publish only to:
  - `beehive/site01/apiary01/hive-001/#`

- It may subscribe only to:
  - `beehive/site01/apiary01/hive-001/config`
  - `beehive/site01/apiary01/hive-001/command`

## 12.3 Raspberry Pi Hardening

Required practices:

- Change default credentials
- Use SSH keys where possible
- Restrict dashboard/admin access to local network
- Enable automatic service restarts
- Back up critical configs
- Monitor disk usage

---

# 13. Reliability and Fault-Tolerance Specification

## 13.1 Node Reliability Requirements

The node shall:

- Use timeouts for sensor read, Wi-Fi connect, and MQTT connect
- Avoid indefinite awake loops
- Use watchdog-safe design
- Recover after reboot
- Buffer unsent messages locally

## 13.2 Buffering Strategy

When publication fails:

- Store payload locally
- Retry oldest buffered payload first on next successful connection
- Keep queue size bounded
- Drop oldest records only if storage is exhausted

## 13.3 Watchdog / Timeout Requirements

Recommended operational limits:

- Sensor read timeout: 2 seconds
- Wi-Fi connect timeout: 10 seconds
- MQTT connect timeout: 5 seconds
- Max wake cycle: 30 seconds

## 13.4 Data Validation Requirements

The firmware shall reject or flag implausible values.

Examples:

- Temperature below -20 °C or above 70 °C
- Humidity below 0% or above 100%
- Battery voltage outside expected chemistry range
- Weight jumps beyond configured threshold

---

# 14. Power Management Specification

## 14.1 Power Strategy

The node shall:

- Spend most time in deep sleep
- Power sensors only when needed
- Minimize Wi-Fi connection time
- Publish a single consolidated payload per wake cycle

## 14.2 Battery-Aware Behavior

The node should adjust sampling frequency based on battery level.

Example policy:

- Healthy battery: frequent sampling
- Medium battery: reduced rate
- Low battery: sparse sampling
- Critical battery: long sleep mode

## 14.3 Final Power Requirement

The final implementation shall demonstrate a repeatable low-power cycle suitable for field operation on battery power.

---

# 15. Implementation Plan

This section defines the recommended execution sequence.

## Phase 1 — Project Preparation

### Objectives
- Finalize scope
- Confirm sensors and node variants
- Define naming scheme
- Approve architecture

### Deliverables
- Final architecture
- Confirmed bill of materials
- Approved MQTT topic design
- Approved InfluxDB schema

## Phase 2 — Gateway Setup

### Tasks
- Install Raspberry Pi OS
- Configure static network access
- Install Mosquitto
- Install InfluxDB
- Install Grafana
- Install ingestion service
- Configure service autostart

### Deliverables
- Local MQTT broker operational
- Database operational
- Dashboard accessible locally
- Services persistent across reboots

## Phase 3 — First ESP Prototype

### Tasks
- Assemble prototype node
- Connect temperature/humidity sensor
- Add battery voltage measurement
- Implement basic firmware
- Publish first telemetry payload

### Deliverables
- Working node prototype
- Data visible in MQTT and InfluxDB
- Deep sleep cycle working

## Phase 4 — End-to-End Validation

### Tasks
- Validate MQTT topic structure
- Validate payload schema
- Validate database ingestion
- Build first dashboard
- Verify reconnection behavior

### Deliverables
- End-to-end working minimal system
- Basic dashboard with live data

## Phase 5 — Add Weight Measurement

### Tasks
- Integrate HX711 and scale hardware
- Implement calibration
- Add filtered weight sampling
- Store weight data in dedicated measurement

### Deliverables
- Calibrated weight monitoring
- Stable stored weight trend

## Phase 6 — Add Sound/Vibration Monitoring

### Tasks
- Select activity sensor
- Compute summary features locally
- Extend MQTT payload and DB schema
- Add dashboard panels

### Deliverables
- Activity monitoring implemented
- No raw continuous streaming required

## Phase 7 — Reliability Hardening

### Tasks
- Add timeouts
- Add watchdog strategy
- Add offline buffering
- Add status/error reporting
- Add retained availability state

### Deliverables
- Fault-tolerant node behavior
- Better resilience to outages

## Phase 8 — Power Optimization

### Tasks
- Measure active and sleep current
- Switch sensor power rails where possible
- Reduce Wi-Fi connection time
- Add battery-aware sampling intervals

### Deliverables
- Power-optimized firmware
- Documented power profile

## Phase 9 — Security Hardening

### Tasks
- Disable anonymous MQTT access
- Create unique per-node users
- Apply ACLs
- Restrict network access
- Harden Raspberry Pi

### Deliverables
- Secure local deployment

## Phase 10 — Field Pilot and Rollout

### Tasks
- Install pilot node
- Run 7–14 day trial
- Review data stability
- Adjust hardware or firmware
- Scale to remaining hives

### Deliverables
- Pilot acceptance report
- Production rollout plan

---

# 16. Acceptance Criteria

This section defines when the project will be considered complete and acceptable.

## 16.1 System-Level Acceptance Criteria

### AC-001 — Local-Only Operation
The system shall operate fully without any cloud service or internet dependency.

**Pass condition:**  
Telemetry continues to flow from node to gateway to database on an isolated local network.

### AC-002 — End-to-End Telemetry
At least one node shall successfully publish telemetry to the Raspberry Pi, and data shall be stored in InfluxDB.

**Pass condition:**  
Sensor values are visible in the database and local dashboard.

### AC-003 — Multi-Hive Readiness
The architecture shall support multiple nodes using unique device and hive identifiers.

**Pass condition:**  
A second node can be added without redesigning the data model or MQTT structure.

### AC-004 — Deep Sleep Operation
The node shall successfully enter and wake from deep sleep according to configured intervals.

**Pass condition:**  
Measured node behavior confirms cyclical wake-publish-sleep operation.

### AC-005 — Offline Recovery
If the MQTT broker or network becomes temporarily unavailable, the node shall recover automatically when service returns.

**Pass condition:**  
After temporary outage, the node resumes publishing without manual intervention.

### AC-006 — Availability Monitoring
Each node shall publish an availability state using MQTT retained messages and last-will behavior.

**Pass condition:**  
Dashboard or MQTT subscriber can distinguish between online and offline states.

### AC-007 — Secure MQTT Access
MQTT publish and subscribe access shall require authentication.

**Pass condition:**  
Unauthenticated clients cannot publish telemetry in production configuration.

### AC-008 — Extendability
The implementation shall permit enabling or adding sensors without redesigning the full system.

**Pass condition:**  
At least one optional sensor module (e.g. weight or vibration) can be added using existing architecture patterns.

## 16.2 Sensor-Level Acceptance Criteria

### AC-009 — Temperature / Humidity
The node shall publish valid temperature and humidity measurements.

**Pass condition:**  
Readings are present and fall within plausible ranges during test.

### AC-010 — Battery Voltage
The node shall publish battery voltage in each measurement cycle.

**Pass condition:**  
Battery voltage appears in telemetry and database.

### AC-011 — Weight (if implemented)
The node shall produce calibrated weight data.

**Pass condition:**  
Known reference weight produces acceptable measurement accuracy after calibration.

### AC-012 — Activity Sensor (if implemented)
The node shall publish sound/vibration summary metrics.

**Pass condition:**  
Metrics appear in the activity measurement and dashboard.

## 16.3 Gateway Acceptance Criteria

### AC-013 — Mosquitto Service Persistence
Mosquitto shall start automatically on Raspberry Pi boot.

### AC-014 — InfluxDB Persistence
InfluxDB shall preserve stored data across reboot.

### AC-015 — Dashboard Availability
Grafana or equivalent local dashboard shall be accessible on the local network.

### AC-016 — Service Recovery
Core services shall restart automatically after Pi reboot.

---

# 17. Acceptance Test Plan

## 17.1 Factory / Bench Tests

### Test T-001 — Sensor Read Test
- Power node
- Read enabled sensors
- Verify plausible values

### Test T-002 — MQTT Publish Test
- Connect node to local Wi-Fi
- Publish telemetry
- Verify broker receives message

### Test T-003 — Database Ingestion Test
- Confirm MQTT ingestion writes expected measurements and fields

### Test T-004 — Deep Sleep Test
- Verify node sleeps and wakes on schedule

### Test T-005 — Availability Test
- Verify `online` on connect and `offline` via LWT when disconnected unexpectedly

## 17.2 Reliability Tests

### Test T-006 — Broker Outage Test
- Stop Mosquitto temporarily
- Confirm node does not hang indefinitely
- Restore broker
- Confirm node resumes operation

### Test T-007 — Wi-Fi Loss Test
- Disable Wi-Fi or remove node from AP range
- Confirm node enters recovery logic
- Restore Wi-Fi
- Confirm resumed publishing

### Test T-008 — Power Recovery Test
- Reboot Raspberry Pi
- Confirm all services return
- Confirm node reconnects automatically

## 17.3 Power Tests

### Test T-009 — Active/Sleep Current Measurement
- Measure node current during wake and deep sleep
- Document average cycle consumption

### Test T-010 — Battery-Aware Behavior
- Simulate low battery threshold
- Verify reduced sampling interval

## 17.4 Field Pilot Tests

### Test T-011 — 7–14 Day Continuous Pilot
- Install one node in real hive environment
- Verify:
  - data continuity
  - battery trend
  - environmental stability
  - enclosure suitability
  - absence of frequent manual intervention

---

# 18. Risks and Mitigations

## Risk R-001 — Poor Wi-Fi Coverage
**Impact:** Lost or delayed telemetry  
**Mitigation:**
- Improve AP placement
- Use external antenna if needed
- Increase offline buffering

## Risk R-002 — Excessive Battery Drain
**Impact:** Short runtime  
**Mitigation:**
- Aggressive deep sleep
- Power-gated sensors
- Reduced sampling frequency
- Solar charging if needed

## Risk R-003 — Sensor Failure / Drift
**Impact:** Bad data quality  
**Mitigation:**
- Add validation rules
- Support calibration
- Log sensor errors
- Design for sensor replacement

## Risk R-004 — SD Card Wear / Storage Failure
**Impact:** Gateway data loss  
**Mitigation:**
- Use SSD
- Use backups
- Reduce unnecessary write load

## Risk R-005 — Moisture / Environmental Damage
**Impact:** Hardware failure  
**Mitigation:**
- IP-rated enclosure
- Cable glands
- Conformal coating where appropriate
- Desiccant / moisture management

---

# 19. Operations and Maintenance

## 19.1 Routine Monitoring

Operators should be able to review:

- Last message time per hive
- Battery voltage trend
- RSSI trend
- Missing data alerts
- Sensor error count
- Disk space on gateway

## 19.2 Maintenance Schedule

### Weekly
- Check dashboards
- Confirm all nodes reporting
- Check battery levels

### Monthly
- Inspect enclosure and cables
- Check scale stability
- Review disk usage and logs
- Back up configs

### Seasonal
- Recalibrate scales
- Inspect batteries
- Verify sensor health
- Update firmware if required

---

# 20. Deliverables

The implementation shall produce the following deliverables:

- Hardware bill of materials
- Wiring diagrams
- ESP firmware source
- Gateway configuration files
- MQTT topic and payload documentation
- InfluxDB schema documentation
- Dashboard configuration
- Test report
- Acceptance report
- Device inventory / calibration sheet

---

# 21. Recommended Initial Bill of Materials (Indicative)

## Sensor Node
- ESP32 development board or custom ESP32 PCB
- SHT31 or BME280
- HX711 + load cells (if weight enabled)
- Accelerometer / vibration sensor or I²S microphone (if activity enabled)
- Battery voltage divider components
- Battery and protection circuitry
- Optional solar charge subsystem
- Outdoor enclosure and connectors

## Gateway
- Raspberry Pi 4
- External SSD
- Reliable power supply
- Optional dedicated Wi-Fi AP/router

---

# 22. Appendix A — Example Node Wake Cycle

```text
1. Wake from deep sleep
2. Initialize minimal services
3. Power sensors
4. Wait for sensor stabilization
5. Read sensors
6. Validate readings
7. Connect to Wi-Fi
8. Connect to MQTT
9. Publish availability = online
10. Publish buffered data if any
11. Publish current telemetry
12. Publish status if scheduled
13. Disconnect
14. Power down sensors
15. Return to deep sleep
```

---

# 23. Appendix B — Example Requirement Traceability Snapshot

| Requirement ID | Description | Implementation Area | Acceptance Test |
|---|---|---|---|
| FR-001 | Measure temperature | Sensor driver | T-001 |
| FR-006 | Publish telemetry via MQTT | MQTT manager | T-002 |
| FR-015 | Store data in InfluxDB | Ingestion service | T-003 |
| FR-010 | Deep sleep operation | Power manager | T-004 |
| FR-012 | Offline buffering | Buffer manager | T-006 / T-007 |
| NFR-011 | MQTT authentication | Mosquitto config | AC-007 |
| NFR-003 | Unattended operation | Whole system | T-011 |

---

# 24. Final Implementation Recommendation

To reduce project risk, the implementation should proceed in this order:

```text
1. Build Raspberry Pi gateway
2. Validate MQTT broker
3. Validate DB ingestion
4. Build first ESP32 node with temperature/humidity + battery
5. Add deep sleep
6. Add dashboards
7. Add reliability layer (timeouts, buffering, watchdog-safe flow)
8. Add weight
9. Add vibration/sound features
10. Harden security and deploy pilot
11. Scale to multiple hives
```

---

# 25. Definition of Done

The project shall be considered complete when all the following are true:

- At least one field-deployed hive node runs autonomously on the local network
- Temperature, humidity, and battery telemetry are stored in InfluxDB
- The gateway remains operational across reboot
- Dashboards show current and historical data locally
- MQTT is authenticated and access-controlled
- Deep sleep and low-power behavior are demonstrated
- The system survives temporary outages and recovers automatically
- The documentation, configuration, and test evidence are handed over
