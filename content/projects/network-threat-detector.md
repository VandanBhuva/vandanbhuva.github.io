---
title: "Kernel Network Threat Detector"
date: 2026-06-01
weight: 2
description: "An XDP/TCX eBPF monitor detecting 7 network attack classes mapped to MITRE ATT&CK, dropping 380,000+ PPS SYN floods at 99.8% in the kernel."
stack: ["C", "Go", "eBPF", "XDP / TCX"]
links:
  code: "https://github.com/VandanBhuva/network-threat-detector"
---

An eBPF network monitor attached at the XDP and TCX hooks that detects and mitigates attacks before packets reach the normal network stack.

## Detection coverage

Seven attack classes, each mapped to MITRE ATT&CK techniques: SYN floods, UDP amplification, port scans, C2 beaconing, DNS tunneling, ARP spoofing, and data exfiltration.

## Results

Kernel-enforced mitigation was benchmarked with `hping3` load tests and verified through eBPF map counters:

| Attack | Load | Dropped |
| --- | --- | --- |
| SYN flood | 380,000+ packets/sec | 99.8% |
| UDP amplification | 3,700+ Mbps | 100% |

Legitimate traffic saw **0% false positive impact** while the floods were running.

## Telemetry

Alerts are emitted as structured JSON with MITRE ATT&CK mappings, ready to ship into a SIEM. Core detection logic is covered by Go unit tests.
