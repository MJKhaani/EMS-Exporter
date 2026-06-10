# EMS-Exporter

Prometheus SNMP Exporter for the **NS-705 SNMP Server Room Controller** — an Environmental Monitoring System (EMS) that monitors temperature, humidity, smoke, power inputs, and triggers alarms for server room environments.

---

## Table of Contents

- [Device Overview](#device-overview)
- [Sensors](#sensors)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
  - [Binary (Native)](#binary-native)
  - [Docker](#docker)
- [Configuration](#configuration)
  - [snmp.yml](#snmps-yml)
  - [Running the Exporter](#running-the-exporter)
- [Prometheus Scrape Job](#prometheus-scrape-job)
- [Verification](#verification)
- [Resources](#resources)

---

## Device Overview

| Property | Value |
|---|---|
| **Model** | EasyNet NS-705B |
| **Manufacturer** | Pardik (پاردیک) — Pardis Engineering Co. |
| **Purpose** | Environmental monitoring for server rooms and data centers |
| **Protocol** | SNMP v1 |
| **Community** | public |
| **IP Address** | 192.168.X.X |
| **Web Interface** | http://192.168.X.X (Default Username/Password: admin/admin) |

### Key Features

- High-precision temperature and humidity control via TDC-23 and THC-22 sensors
- Configurable setpoints for temperature and humidity thresholds
- Automatic activation of reserve coolers on temperature rise
- Support for 2–3 external sensor zones per device
- Smoke, water leak, and power failure detection per zone
- Multiple dry-contact outputs for remote power cycling (reset/reboot) of server equipment
- Audible and visual alarm triggering
- Automatic ping-based server/router health check with auto-reboot on hang detection
- Full SNMP-based reporting to network monitoring tools (e.g. PRTG)
- Web-based panel for configuration and network parameter changes
- Password-protected access

### Device Photo

![NS-705 Device](resources/NS-705.jpg)

See [resources/](resources/) for the full product datasheet (`NS-705_SNMP Server room controller.pdf`).

---

## Sensors

The NS-705 exposes the following sensors via SNMP, currently monitored in PRTG under **ROOT → Local Probe → Environmental Parameters** (192.168.X.X):

| # | Sensor Name | OID | PRTG Value | Description |
|---|---|---|---|---|
| 1 | Temp 1 (Rack1-Front) | `1.3.6.1.4.1.61.1.1.2.1.1.87` | 28 °C | Temperature at Rack 1 front |
| 2 | Temp 2 (Rack2-Rear) | `1.3.6.1.4.1.61.1.1.2.1.1.88` | 31 °C | Temperature at Rack 2 rear |
| 3 | Temp 3 (Rack3-Rear) | `1.3.6.1.4.1.61.1.1.2.1.1.89` | 32 °C | Temperature at Rack 3 rear |
| 4 | Temp 4 (Rack5-Front) | `1.3.6.1.4.1.61.1.1.2.1.1.90` | 25 °C | Temperature at Rack 5 front |
| 5 | THC | `1.3.6.1.4.1.61.1.1.2.1.1.91` | 26 | Temperature-Humidity Composite sensor |
| 6 | Smoke | `1.3.6.1.4.1.61.1.1.2.1.1.83` | 0 (normal) | Smoke detector (0 = no smoke) |
| 7 | Power 1 | `1.3.6.1.4.1.61.1.1.2.1.1.84` | 1 (active) | Main power input status |
| 8 | Power 2 (Input Generator) | `1.3.6.1.4.1.61.1.1.2.1.1.85` | 0 (inactive) | Generator power input status |
| 9 | Input Zone 1 | `1.3.6.1.4.1.61.1.1.2.1.1.81` | 0 | External sensor zone 1 |
| 10 | Input Zone 2 | `1.3.6.1.4.1.61.1.1.2.1.1.82` | 0 | External sensor zone 2 |

All PRTG sensors use `snmpcustom` (raw OID) type. OIDs discovered from PRTG at `https://mon.ibagher.ir` (device: Environmental Parameters, objid 5185).

### Hardware Notes

- **Enterprise OID:** `1.3.6.1.4.1.61` (registered to Merit Network)
- **MAC OUI:** `00:FE:A7` (private assignment)
- **Web server:** lwIP/uIP-based embedded stack running on a custom MCU
- **SNMP agent:** Minimal implementation — responds only to direct GET requests on configured OIDs; does not support WALK/BULK or System MIB queries

---

## Architecture

```
┌─────────────────────┐         SNMP v1          ┌──────────────────────┐
│   NS-705 EMS Device │ ◄──────────────────────► │   snmp_exporter       │
│     192.168.X.X     │    community: public     │   :9116               │
│                     │                          │   (this project)      │
│  • 4× Temp sensors  │                          └──────────┬───────────┘
│  • THC sensor       │                                     │ HTTP
│  • Smoke detector   │                                     ▼
│  • 2× Power inputs  │                          ┌──────────────────────┐
│  • Dry-contact I/O  │                          │   Prometheus          │
└─────────────────────┘                          │   (scrape :9116)      │
                                                 └──────────────────────┘
```

---

## Prerequisites

- **snmp_exporter** binary or Docker image
- Network reachability to `192.168.X.X:161/udp` (same L2/L3 network or routed)
- Prometheus server (for scrape configuration)

---

## Installation

### Binary (Native)

```bash
# Download the latest release
wget https://github.com/prometheus/snmp_exporter/releases/download/v0.30.1/snmp_exporter-0.30.1.darwin-amd64.tar.gz
tar xzf snmp_exporter-0.30.1.darwin-amd64.tar.gz
sudo mv snmp_exporter-0.30.1.darwin-amd64/snmp_exporter /usr/local/bin/
```

### Docker

```bash
docker run -d \
  --name snmp_exporter_env \
  --restart unless-stopped \
  -p 9116:9116 \
  -v $(pwd)/snmp.yml:/etc/snmp_exporter/snmp.yml:ro \
  prom/snmp-exporter:v0.30.1 \
  --config.file=/etc/snmp_exporter/snmp.yml
```

---

## Configuration

### snmp.yml

The snmp.yml configuration file uses the `auths` + `modules` structure required by snmp_exporter v0.30.x. All OIDs have been verified against the live device via PRTG at `https://mon.ibagher.ir`.

The device only supports direct SNMP GET (not WALK/BULK), so the module uses `get` with explicit OID list instead of `walk`.

```yaml
auths:
  ns705_v1:
    version: 1
    community: public

modules:
  ns705:
    get:
    - 1.3.6.1.4.1.61.1.1.2.1.1.81
    - 1.3.6.1.4.1.61.1.1.2.1.1.82
    - 1.3.6.1.4.1.61.1.1.2.1.1.83
    - 1.3.6.1.4.1.61.1.1.2.1.1.84
    - 1.3.6.1.4.1.61.1.1.2.1.1.85
    - 1.3.6.1.4.1.61.1.1.2.1.1.87
    - 1.3.6.1.4.1.61.1.1.2.1.1.88
    - 1.3.6.1.4.1.61.1.1.2.1.1.89
    - 1.3.6.1.4.1.61.1.1.2.1.1.90
    - 1.3.6.1.4.1.61.1.1.2.1.1.91
    metrics:
    - name: envInputZone1
      oid: 1.3.6.1.4.1.61.1.1.2.1.1.81
      type: gauge
      help: External sensor zone 1 status.
    - name: envInputZone2
      oid: 1.3.6.1.4.1.61.1.1.2.1.1.82
      type: gauge
      help: External sensor zone 2 status.
    - name: envSmokeStatus
      oid: 1.3.6.1.4.1.61.1.1.2.1.1.83
      type: gauge
      help: Smoke detector status. 0=no smoke, 1=smoke detected.
      enum_values:
        0: no_smoke
        1: smoke_detected
    - name: envPower1Status
      oid: 1.3.6.1.4.1.61.1.1.2.1.1.84
      type: gauge
      help: Main power input status. 1=active, 0=inactive.
      enum_values:
        0: inactive
        1: active
    - name: envPower2GeneratorStatus
      oid: 1.3.6.1.4.1.61.1.1.2.1.1.85
      type: gauge
      help: Generator power input status. 1=active, 0=inactive.
      enum_values:
        0: inactive
        1: active
    - name: envTemp1Rack1Front
      oid: 1.3.6.1.4.1.61.1.1.2.1.1.87
      type: gauge
      help: Temperature at Rack 1 Front in Celsius.
    - name: envTemp2Rack2Rear
      oid: 1.3.6.1.4.1.61.1.1.2.1.1.88
      type: gauge
      help: Temperature at Rack 2 Rear in Celsius.
    - name: envTemp3Rack3Rear
      oid: 1.3.6.1.4.1.61.1.1.2.1.1.89
      type: gauge
      help: Temperature at Rack 3 Rear in Celsius.
    - name: envTemp4Rack5Front
      oid: 1.3.6.1.4.1.61.1.1.2.1.1.90
      type: gauge
      help: Temperature at Rack 5 Front in Celsius.
    - name: envTHC
      oid: 1.3.6.1.4.1.61.1.1.2.1.1.91
      type: gauge
      help: Composite Temperature-Humidity sensor value.
    retries: 3
    timeout: 5s
```

### Running the Exporter

```bash
# Start manually (foreground)
snmp_exporter \
  --config.file=snmp.yml \
  --web.listen-address=127.0.0.1:9116

# Run in background
nohup snmp_exporter \
  --config.file=snmp.yml \
  --web.listen-address=127.0.0.1:9116 \
  > /dev/null 2>&1 &
```

---

## Prometheus Scrape Job

Add the following to your `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'env_sensors'
    static_configs:
      - targets:
          - 192.168.X.X
    metrics_path: /snmp
    params:
      module: [ns705]
      auth: [ns705_v1]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9116
```

---

## Verification

### 1. Check the exporter is running

```bash
curl -s http://127.0.0.1:9116/metrics | head -5
```

### 2. Test SNMP collection manually

```bash
# Open in browser or use curl:
# http://127.0.0.1:9116/snmp?module=ns705&target=192.168.X.X&auth=ns705_v1

curl -s "http://127.0.0.1:9116/snmp?module=ns705&target=192.168.X.X&auth=ns705_v1"
```

### 3. Module name not found

If you see `"Unknown auth"` error, make sure the `auth` parameter matches the name under `auths:` in snmp.yml (i.e., `ns705_v1`).

### 4. Expected metrics

Once the device is reachable and OIDs are correct, you should see metrics like:

```
envTemp1Rack1Front{instance="192.168.X.X"} 28
envTemp2Rack2Rear{instance="192.168.X.X"} 31
envTemp3Rack3Rear{instance="192.168.X.X"} 32
envTemp4Rack5Front{instance="192.168.X.X"} 25
envTHC{instance="192.168.X.X"} 26
envSmokeStatus{instance="192.168.X.X"} 0
envPower1Status{instance="192.168.X.X"} 1
envPower2GeneratorStatus{instance="192.168.X.X"} 0
envInputZone1{instance="192.168.X.X"} 0
envInputZone2{instance="192.168.X.X"} 0
```

---

## Resources

| File | Description |
|---|---|
| [resources/NS-705.jpg](resources/NS-705.jpg) | Device photo |
| [resources/NS-705_SNMP Server room controller.pdf](resources/NS-705_SNMP%20Server%20room%20controller.pdf) | Product datasheet (3 pages, Persian) |
| [snmp.yml](snmp.yml) | snmp_exporter configuration |
| [mibs/](mibs/) | Downloaded MIB files |

---

## See Also

- [prometheus/snmp_exporter](https://github.com/prometheus/snmp_exporter) — Official repo
- [SNMP Exporter Generator](https://github.com/prometheus/snmp_exporter/tree/main/generator) — Web-based snmp.yml generator
- [PRTG Network Monitor](https://www.paessler.com/prtg) — The existing monitoring system
