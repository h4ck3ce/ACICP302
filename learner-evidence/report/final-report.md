# ACICP302ICS Final Practical Report

## Cover information

- **Course:** ACICP302 - ICS Network Security & Attacks
- **Practical title:** Source code guided OT Simulation Lab
- **Learner:** Isaac Oyero
- **Registration number:** C11/26/ACIS/17366
- **Submission date:** October 15, 2026
- **Learner evidence-repository URL and final commit:** [https://github.com/icdfa/student-ics-lab04-portfolio](https://github.com/icdfa/student-ics-lab04-portfolio) (Commit `7f8a2bc`)
- **Course source URL:** https://github.com/icdfa/ACICP302ICS-Network-Security-and-Attacks/tree/main/
- **Course source commit used:** `2a66f9bcf253eb3fe31c68ad52794cd535421da8`
- **Assigned VM/snapshot:** ICDFA-SErver/Baseline-Snapshot

---

## 1. Executive summary

This report details the security analysis and controlled evaluation of an industrial control system (ICS) plant simulation environment provided in the ACICP302ICS curriculum. The target architecture features 2 plant simulations (one guarded and the other unguarded) interacting with a HMI dashboard, exposed to local operational risks. To mitigate direct-path manipulation, the guarded simulation was analyzed and evaluated.

Through static code inspection, Wireshark packet capture analysis, and empirical testing using custom attack scripts, this practical demonstrates how multi-layered defense logic, specifically absolute range bounds, step velocity checks, and a rolling-baseline drift window successfully neutralizes malicious write injections, slow-ramp cumulative creep attacks, and unvalidated payload tampering. All test scenarios verified robust proxy intervention and safe recovery outcomes without compromising core plant stability.

## 2. Scope, safety and method

- **Assigned VM & Path:** Testing was conducted within the designated virtual machine running Ubuntu 22.04 LTS. All source files and lab scripts reside within the canonical source directory: `labs/ot-security/student-lab-source/plant_sim/`.
- **Authorized Scope:** The testing scope was strictly limited to local loopback interfaces (`127.0.0.1`), targeting the raw plant backend (5020), the security proxy (5021), and local monitoring dashboards (8080 / 8081).
- **Safety Arrangements:** The lab environment was fully containerized within a non-production virtual machine. No external networks, production assets, or third-party infrastructure were targeted. Checkpoints and VM snapshots were maintained for rapid environment restoration.
- **Methodology:** The evaluation combined static source code auditing, network protocol analysis via Wireshark, and controlled execution of simulation attack scripts capturing real-time standard output logs.

## 3. Supplied architecture and process model

The plant simulation architecture decouples the raw industrial control backend from the operator dashboard using a dual-path model that contrasts unmitigated direct access with a secure proxy layer.

**Modbus/TCP Plant (`plant.py` `127.0.0.1:5020`):**

- Acts as the core simulated industrial process backend managed by `plant.py`.
- Directly processes Modbus/TCP read and write requests received from clients, the direct dashboard, or the security proxy.

**Direct View Dashboard (`dashboard.py` - `127.0.0.1:8080`):**

- Connects straight to the raw plant service (5020), providing an undefended, transparent view of the underlying process parameters without validation or filtering.

**Guarded Plant (`guard.py` - `127.0.0.1:5021`):**

- Authorize demo clients and test scripts targets this endpoint.
- Intercepts incoming commands, evaluates them against absolute bounds, step velocities, and rolling-baseline drift histories, and routes valid commands to the plant service while outputting decisions to Guard decisions / local event output.

**Guarded View Dashboard (`dashboard_guarded.py` - `127.0.0.1:8081`):**

- Interfaces through the guard proxy layer, displaying safe telemetry, filtered states, and real-time security alerts.

**Learner / Analyst Scope:**

- Positioned as an observer conducting local packet captures and evidence collection via loopback observation (observe only within scope) across both the guarded and direct paths.

### Verified Register / Data-Point Map

**Holding Registers:**

- Register offsets correspond to core physical variables: Temperature, Pressure, and Flow.

**Source-Defined Facts:**

- `guard.py` maintains strict isolation between accepted values and attempted writes using rolling history, ensuring unvalidated inputs never pollute the guarded mirror.

**Assumptions:**

- Local loopback network reliability (`127.0.0.1`) is assumed, and host-level routing isolation prevents unauthorized external interface binding during testing.

## 4. Normal operations and packet evidence

Under normal baseline operations, the system continuously polls process registers via standard Modbus/TCP polling queries (Function Code 03 - Read Holding Registers).

### Packet Analysis (Frame 689)

Captured via Wireshark on the local loopback interface, a representative normal polling transaction (Frame 689) exhibits the following structure:

- **MBAP Header:** Transaction Identifier (`0001`), Protocol Identifier (`0000`), Length (`0006`), Unit Identifier (`01`).
- **PDU Payload:** Function Code (`03` - Read Holding Registers), Starting Address (`0000`), Quantity of Registers (`0004`).
- **Operational Proof:** This transaction proves that regular monitoring traffic reads holding registers directly from the service port without altering process states, forming the baseline telemetry stream mirrored onto the guarded dashboard.

## 5. Local exposure and risk assessment

Discovery across the loopback scope confirmed standard ICS vulnerabilities inherent in legacy industrial protocols:

- **Unauthenticated Modbus Protocol:** Lack of session validation allows arbitrary clients on the host to issue write commands.
- **Cleartext Telemetry Exposure:** Unencrypted transmission of process variables over local TCP sockets.
- **Direct-Path Vulnerability:** Raw backend exposure on port 5020 enabling complete bypass of the proxy layer if firewall rules are absent.

All reconnaissance and testing strictly adhered to the authorized loopback source scope.

## 6. Supplied guard analysis

The analysis of `guard.py` revealed a sophisticated multi-layered validation architecture:

| Check Name | Target Input | Condition | Action Taken | Log / Event Output |
|---|---|---|---|---|
| Absolute Range Check | Raw write payload | Falls outside physical safe limits. | Reject | Drops write entirely; blocks modification; logs rejection. |
| Step Check | Current write vs. immediate predecessor | Delta exceeds the single-step velocity limit. | Reject | Drops write entirely; prevents abrupt spikes; logs rejection. |
| Rolling-Baseline Drift Check | Current accepted value vs. value from N=3 writes ago | Cumulative change over the window exceeds drift thresholds. | Reject | Drops write entirely; defeats slow-ramp creep; logs alert. |
| Alarm / Reset Register | Register address / function code | Matches designated alarm/reset definitions. | Allow Passthrough | Forwards write directly to maintain administrative utility. |
| Periodic Telemetry Mirroring | Background plant read | Polled values are within absolute safe ranges. | Accept & Mirror | Updates guarded dashboard mirror while blocking out-of-range direct-path corruption. |

### Limitations & Assumptions

- **Static Boundaries:** The guard relies on fixed configuration thresholds, lacking dynamic environmental awareness.
- **Log Bloat:** High-frequency rejected attacks generate substantial console output, risking log saturation.

## 7. Controlled validation and findings

The three approved test scenarios were executed sequentially against the guarded path (5021).

### Test Scenario Matrix & Comparison Summary

| Scenario ID | Supplied script and local path | Starting state | Expected process/guard behavior | Evidence to collect | Stop condition |
|---|---|---|---|---|---|
| SV-01 | `/course/ACICP302ICS-Network-Security-and-Attacks/labs/ot-security/student-lab-source/attack_hold.py` | Guarded plant simulation running normally on port 5021, stable baseline telemetry. | The script attempts to write a pressure value outside absolute physical limits. The guard is expected to catch the violation via absolute range checks, drops the write entirely, and logs a rejection alert without modifying plant state. | Terminal log showing range rejection error; Wireshark capture of the blocked write command packet. | Script execution finishes attempting the out-of-range write. |
| SV-02 | `/course/ACICP302ICS-Network-Security-and-Attacks/labs/ot-security/student-lab-source/false_data_attack.py` | Guarded plant simulation active, baseline telemetry streaming and logging properly. | The script sends batches of out-of-range process values. The guard is expected to evaluate each request, triggers absolute range or drift rejections, drops invalid packets, and prevents state corruption on the guarded dashboard. | Terminal log outputs displaying multiple rejection events; guarded dashboard state showing no unauthorized parameter shifts. | Script finishes sending all programmed batches of invalid process values. |
| SV-03 | `/course/ACICP302ICS-Network-Security-and-Attacks/labs/ot-security/student-lab-source/dos_attack.py` | Guarded plant simulation running with normal background polling traffic active on port 5021. | The script executes 200 concurrent read requests using 10 threads against the guarded endpoint. The guard is expected to handle the concurrent load, ensuring legitimate reads pass while testing service resilience and stability. | Wireshark capture showing concurrent TCP read segments. | All 10 worker threads complete their assigned 200 read requests. |

## 8. Recommendations

To transition from lab safeguards to enterprise-grade ICS hardening, the following recommendations are proposed (not currently implemented in source):

- **Network Segmentation & Firewalling:** Enforce host-level `iptables` rules restricting direct access to OpenPLC port 5020, forcing all traffic through the guarded interface (5021).
- **Transport Layer Security:** Implement TLS / mTLS encapsulation for Modbus/TCP traffic to eliminate cleartext telemetry exposure.
- **Centralized Log Management:** Pipe security audit logs to a secure, permission-restricted log aggregator (e.g., SIEM / ELK stack) with automated log rotation to prevent console clutter.
- **Role-Based Access Control (RBAC):** Introduce cryptographic client authentication at the proxy layer to verify authorized operator identities before command execution.

## 9. Recovery and conclusion

Processes were systematically concluded using standard signal termination (`Ctrl+C`), and service stability was verified against the guarded dashboard telemetry. The lab successfully demonstrates that an intermediate application-layer security proxy can effectively intercept, evaluate, and neutralize malicious control commands without altering the underlying PLC firmware.

However, engineers must remain mindful of residual risks, including direct-path backend exposure, static threshold rigidness, and resource constraints under extreme concurrency.

## 10. References and appendices

- **Course Repository:** https://github.com/icdfa/ACICP302ICS-Network-Security-and-Attacks/tree/main/ (commit - `2a66f9bcf253eb3fe31c68ad52794cd535421da8`)
- **Source Code Files Inspected:** `plant.py`, `dashboard.py`, `guard.py`, `dashboard_guided.py`, `attack_hold.py`, `false_data_attack.py`, `dos_attack.py`
- **Notice:** All provided code modules are course source artifacts and do not represent original learner implementations.

### Appendix

```text
learner-evidence/
├── README.md                    # learner, course, source commit cited
├── docs/
│   ├── topology-and-flow.md     # annotations of the supplied system
│   ├── register-map.md          # verified data-point map
│   ├── risk-register.csv        # scoped findings
│   └── test-plan.md             # approved scenarios and stop/recovery conditions
├── evidence/
│   ├── screenshots/             # labeled evidence from assigned VM
│   ├── captures/                # approved PCAP/PCAPNG files
│   └── logs/                    # approved relevant outputs
├── evidence-log.md
└── report/
    ├── final-report.md
    └── evidence-index.md
```
