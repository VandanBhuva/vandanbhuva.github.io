---
title: "eBPF Ransomware Behavior Monitor"
date: 2026-11-01
weight: 1
featured: true
description: "A Linux EDR agent built on eBPF LSM hooks that detects ransomware by behavior and kills it mid-attack. Preserved 99.2% of files in a 1,000-file encryption outbreak."
stack: ["C", "Go", "eBPF", "LSM hooks"]
links:
  code: "https://github.com/VandanBhuva/ebpf-malware-monitor"
---

> Current main focus. This project is under active development.

A Linux endpoint detection and response (EDR) agent that watches process behavior from inside the kernel using eBPF programs attached to Linux Security Module (LSM) hooks. Instead of matching signatures, it scores what a process is *doing* and terminates it while the attack is still in progress.

## Results

- **99.2% of files preserved** in a 1,000-file encryption outbreak.
- **All 4 attack patterns contained** across 12 test runs.
- **False positives on benign workloads cut from 5 to 1** (`tar`, `dpkg`, `mv`).
- **16 MB idle memory** for the full agent.

## Detection engine

The core is a stateful scoring engine that correlates several weak signals over sliding time windows, so no single noisy indicator triggers a kill on its own:

- **Write velocity:** how fast a process is rewriting files.
- **Entropy:** whether written data looks encrypted.
- **Write-then-delete pairs:** the classic "write encrypted copy, remove original" pattern.
- **Ransom-note spread:** the same new file appearing across many directories.

Correlating these is what let the engine tell real ransomware apart from legitimate bulk file operations like archive extraction and package installs.

## Hardening against evasion

A detector is only useful if the malware can't sidestep it or turn it against innocent processes. I closed three evasion paths:

1. **Process name spoofing:** a malicious binary renaming itself to look like a trusted tool.
2. **PID reuse:** tricking the agent into killing an unrelated process that inherited a recycled PID.
3. **Pre-started malware:** processes that were already running before the agent loaded.

The fix was to pin each process to the **SHA-256 identity of its executable** inside the kernel, and to deliver kill signals through **verified pidfds** rather than raw PIDs, so a signal always reaches the process that was actually scored.

{{< todo >}}Optional: add an architecture diagram (kernel ↔ user-space data flow), a short demo GIF of a kill mid-encryption, and a "what's next" section.{{< /todo >}}
