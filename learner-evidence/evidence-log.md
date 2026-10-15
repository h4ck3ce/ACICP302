## Execution Log Table

| Time | Starting state | Exact authorized action / supplied script | Observation/result | Evidence reference | Analysis/conclusion | Stop/recovery action |
|---|---|---|---|---|---|---|
| 14:05 PM WAT | Guarded plant simulation running normally on port 5021, stable baseline telemetry streaming. | **Baseline Capture:** Initiated Wireshark capture on local loopback interface and inspected normal polling query transactions. | Captured recurring Modbus/TCP read requests (Function Code 03) polling registers successfully without modifying state. | Wireshark Frame 689 (`baseline_capture.pcapng`) | Confirms normal operational polling traffic flows cleanly and establishes baseline metrics. | None required (passive observation). |
| 14:12 PM WAT | Guarded plant simulation active, baseline telemetry operational. | **SV-01 Execution:**<br><br>`python3 attack_hold.py` targeting guarded port 5021. | Script attempted out-of-range pressure write. The guard intercepted the payload, and dropped it entirely. The plant state remained unmodified. | Terminal log rejection output; Wireshark capture of blocked write packet. | Absolute range checks successfully block hazardous setpoint modifications at the proxy layer. | Script finished execution; no plant recovery needed as the state was not corrupted. |
| 14:18 PM WAT | Guarded plant simulation active, baseline telemetry active on guarded path. | **SV-02 Execution:**<br><br>`python3 false_data_attack.py` targeting guarded port 5021. | Script sent batches of invalid, out-of-range process values. Guard evaluated each batch, triggered absolute/drift rejections, and prevented dashboard corruption. | Terminal output showing sequential rejection events; guarded dashboard display showing stable parameters. | Multi-batch malicious payloads are effectively neutralized by proxy logic and validation checks. | Reset test script state; plant backend required no manual intervention. |
| 14:25 PM WAT | Guarded plant simulation running with normal background polling active on port 5021. | **SV-03 Execution:**<br><br>`python3 dos_attack.py` targeting guarded port 5021. | Script executed 200 concurrent read requests using 10 threads. Proxy processed concurrency load while maintaining valid read passthrough. | Terminal execution summary output; concurrent TCP read packet trace in Wireshark. | Proxy handles standard multi-threaded polling loads stably without crashing under moderate concurrency. | Proxy service verified stable; no manual restart required. |

## Safety Declaration

I confirm that this evidence was gathered only in the assigned ICDFA isolated Ubuntu simulation using the canonical student source, that no real or external system was targeted, and that the stated stop and recovery actions were completed. The course-provided source is attributed and is not represented as learner-authored code.

**Learner signature:** Isaac Oyero

**Date:** October 15, 2026
