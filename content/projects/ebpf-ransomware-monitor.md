---
title: "eBPF Ransomware Behavior Monitor"
date: 2026-11-01
weight: 1
featured: true
description: "A Linux EDR agent built on eBPF LSM hooks that detects ransomware by behavior, kills it mid-attack, and streams telemetry to a Datadog SIEM. Over 97% of files preserved against 1,000-file outbreaks."
stack: ["C", "Go", "eBPF", "LSM", "Linux kernel"]
links:
  code: "https://github.com/VandanBhuva/ebpf-malware-monitor"
---

> Current main focus. This project is under active development.

A Linux endpoint detection and response (EDR) agent that watches process behavior from inside the kernel using eBPF programs attached to Linux Security Module (LSM) hooks. It scores each process on how it behaves rather than on signatures, kills it once the score crosses a threshold, and sends telemetry to a Datadog SIEM.

## Results

- **Over 97% of files preserved** against 1,000-file ransomware outbreaks.
- **Zero false positives** on heavy benign I/O workloads such as `tar` and `gzip`.
- **~1.4 s maximum latency** under stress tests of 125,000 events per burst.

## Detection engine

The scoring engine combines four signals over sliding time windows. One signal on its own never triggers a kill.

- **Write velocity:** how fast a process is rewriting files.
- **Entropy:** whether written data looks encrypted.
- **Write-then-delete pairs:** the classic "write encrypted copy, remove original" pattern.
- **Ransom-note spread:** the same new file appearing in 3 or more directories.

`tar` and `gzip` also write a lot of high-entropy data quickly. Tightening the write-delete check and requiring ransom notes in 3+ directories brought their false positives to zero.

## Hardening against evasion

Ransomware can try to hide from the agent or trick it into killing the wrong process. Two changes block that:

- **Binary identity, not names:** the agent records each process's definitive executable identity (device and inode) at `bprm_creds_for_exec` and keeps it in **task-local storage**, Renaming the binary or reusing a PID doesn't change that identity.
- **Processes started before the agent:** **BPF task iterators** seed state for every process that was already running before the agent loaded.

## Performance under load

The kernel-to-user-space event queue uses **kernel-side drop counters** and **per-worker bounds**, When events flood in, the agent drops some and counts them instead of stalling.
