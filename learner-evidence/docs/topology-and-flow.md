# Supplied Simulation Topology and Register Map

## 1. Source and scope

- **Course source URL:** `/course/ACICP302ICS-Network-Security-And-Attacks/labs/ot-security/student-lab-source`
- **Source commit used:** `2a66f9bcf253eb3fe31c68ad52794cd535421da8`
- **Canonical path:** `labs/ot-security/student-lab-source/`
- **Assigned Ubuntu VM/snapshot:** ICDFA Server/ Baseline_Snapshot
- **Authorized network scope:** loopback (`127.0.0.1`) only

## 2. Annotated system topology

The system architecture features a dual-path layout dividing raw unmitigated access from secured proxy-mediated flows:

- **Modbus/TCP Plant Service (`plant.py` - `127.0.0.1:5020`):** Manages the underlying simulated industrial process state and register memory.
- **Guard Plant (`guard.py` - `127.0.0.1:5021`):** Intercepts write requests, evaluates payloads against absolute and rolling drift rules, outputs decisions to local logs, and forwards validated commands.
- **Direct View Dashboard (`dashboard.py` - `127.0.0.1:8080`):** Connects straight to the raw plant service to display unvalidated telemetry.
- **Guarded View Dashboard (`dashboard_guarded.py` - `127.0.0.1:8081`):** Interfaces through the proxy layer to present safe telemetry and active security event notifications.
- **Authorized demo client & Attack scripts:** Initiate Modbus/TCP commands targeting either the raw plant backend or the security proxy.

## 3. Supplied component inventory

| Component/source file | Supplied role | Local endpoint (if applicable) | Learner observation/source reference | Limitation or assumption |
|---|---|---|---|---|
| `plant.py` | Simulates industrial process logic and holds process registers. | `127.0.0.1:5020` | Wireshark packet captures and direct backend telemetry audits. | Lacks native protocol authentication or authorization (FC05/FC16). |
| `guard.py` / Validation Proxy | Intercepts, validates, and filters incoming Modbus/TCP traffic. | `127.0.0.1:5021` | Terminal execution logs and rejection event tracking (SV-01, SV-02, SV-03). | Relies on static configuration thresholds and single-process Python performance limits. |
| `dashboard.py` / Direct View HMI | Renders live process parameters directly from the backend. | `127.0.0.1:8080` | Direct HMI interface inspection and state observation. | Exposes unmitigated control views without security alert integration. |
| `dashboard_guarded.py` / Guarded HMI | Displays filtered operational parameters and guard logs. | `127.0.0.1:8081` | Guarded dashboard monitoring during attack scenarios. | Dependent on proxy uptime and mirrored background synchronization. |

## 4. Verified Modbus/TCP data-point map

| Data point | Register/address | Data type/scale | Unit | Observed normal range | Alarm/unsafe meaning | Source/evidence reference |
|---|---|---|---|---|---|---|
| Process Temperature | Register 00 | Integer / Scaled x10 | °C | 600-750 | Exceeding thermal safe threshold or rapid artificial spike (>2.0°C delta). | `guard.py` absolute threshold rules & Wireshark capture logs |
| System Pressure | Register 01 | Integer / Scaled x10 | bar | 300-400 | Out-of-bounds pressure setpoints capable of simulating physical vessel stress. | `attack_hold.py` execution output & logs |
| Flow Rate | Register 02 | Integer / Scaled x10 | Nm3/h | 500-1500 | Uncontrolled flow disruption or valve manipulation attempt. | Baseline operational polling queries (Frame 689) |

## 5. Normal data flow and baseline

Under normal operational conditions, monitoring dashboards (`dashboard.py` and `dashboard_guarded.py`) and periodic polling clients issue standard Modbus/TCP read requests (Function Code 03 - Read Holding Registers) to fetch live process values from the plant simulation.

- **Direct Path Flow:** `dashboard.py` polls port 5020 directly, receiving raw, unfiltered process telemetry.
- **Guarded Path Flow:** `dashboard_guarded.py` and authorized clients query port 5021, where `guard.py` executes background polling synchronization and validates state integrity before returning safe telemetry.
- **Baseline Evidence:** Normal baseline traffic maintains steady state readings without triggering proxy rejections, confirmed by loopback packet captures (e.g., Wireshark Frame 689) showing clean request-response transactions across local loopback ports.
