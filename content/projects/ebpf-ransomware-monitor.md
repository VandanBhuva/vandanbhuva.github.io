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

A Linux endpoint detection and response (EDR) agent that watches process behavior from inside the kernel using eBPF programs attached to Linux Security Module (LSM) hooks. Instead of matching signatures, it scores what a process is *doing*, terminates it while the attack is still in progress, and streams telemetry to a Datadog SIEM.

## Results

- **Over 97% of files preserved** against 1,000-file ransomware outbreaks.
- **Zero false positives** on heavy benign I/O workloads such as `tar` and `gzip`.
- **~1.4 s maximum latency** under stress tests of 125,000 events per burst.

## Detection engine

A stateful scoring engine correlates several weak signals over sliding time windows, so no single noisy indicator triggers a kill on its own:

- **Write velocity:** how fast a process is rewriting files.
- **Entropy:** whether written data looks encrypted.
- **Write-then-delete pairs:** the classic "write encrypted copy, remove original" pattern.
- **Ransom-note spread:** the same new file appearing across **3 or more directories**.

Refining the write-delete correlation and requiring ransom notes in several directories is what separated real ransomware from legitimate archive and compression jobs.

## Hardening against evasion

A detector is only useful if malware can't sidestep it or turn it against innocent processes:

- **Binary identity, not names:** the agent records each process's definitive executable identity (device and inode) at `bprm_creds_for_exec` and keeps it in **task-local storage**, so renaming a binary or recycling a PID doesn't fool it.
- **No blind spot at startup:** **BPF task iterators** seed state for every process that was already running before the agent loaded.

## Performance under load

The kernel-to-user-space event queue uses **kernel-side drop counters** and **per-worker bounds**, so a flood of events degrades gracefully and measurably instead of stalling detection.
