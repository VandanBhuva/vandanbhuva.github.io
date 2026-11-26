---
title: "eBPF ransomware monitor, part 1: getting events out of the kernel"
date: 2026-11-25T10:01:00-05:00
weight: 1
series: "ebpf-build-log"
tags: ["ebpf", "linux", "edr"]
description: "Starting a ransomware detector with syscall tracepoints, a BPF ring buffer and a Go agent."
---

*July 16 to 20, 2026.* This series follows how I built the [eBPF Ransomware Behavior Monitor](/projects/ebpf-ransomware-monitor/), a Linux agent that detects ransomware from its behavior and stops it mid-attack. It's reconstructed from the project's 63 commits, README and benchmarks (July 16 to November 11), plus my own notes. The project ran in two bursts: a fast build in July and August, then a measure-and-harden pass from late October.

| Dates | Part | Outcome |
| --- | --- | --- |
| Jul 16 to 20 | 1. First hooks, ring buffer, Go agent | Events flowing from kernel to a Go agent |
| Jul 24 to 27 | 2. LSM hooks, detection engine, kill switch | Scoring engine with SIGSTOP, snapshot, SIGKILL |
| Aug 13 to Sep 8 | 3. Logging, Datadog, worker pool, first benchmarks | JSON logs into Datadog, 32-worker pipeline |
| Oct 25 to 28 | 4. Re-measurement | Found 77% of events lost silently under load |
| Oct 30 to Nov 9 | 5. Process identity, PID reuse, windowed indicators | Name spoofing and PID reuse no longer evade or misfire |
| Nov 9 to 11 | 6. Content check rewrite, exclusion audit | False positives down from 6/12 runs to 0/24 |

I had no summer internship, so the summer went into this project. From August 19 to October 25 I paused it to focus on the fall semester.

## Picking the project

In early July I started reading about eBPF: what it is, what it does, and how it's used for security. I wanted a security project built on it. Claude suggested a few ideas, and I picked ransomware detection rather than going deeper into network security, which I'd already touched with a small [eBPF network threat detector](/projects/network-threat-detector/). I wrote this first version by hand.

## The plumbing

I started with the toolchain, a `vmlinux.h` generated from the running kernel's BTF, and instructions for compiling and loading a BPF program. My first README planned kprobes or tracepoints. I went with syscall tracepoints, because they're the easiest way to watch file activity.

- **Jul 18:** tracepoints on `sys_enter_openat`, then `sys_enter_renameat2` to see file modifications.
- **Jul 19:** a BPF ring buffer to carry events to user space.
- **Jul 20:** the Go agent, using `bpf2go` bindings to load the program and poll the ring buffer.

## First bug

My first real bug was in string handling. Filenames came across as fixed-size C arrays, and Go printed whatever followed the terminator. Parsing null-terminated strings properly fixed it. A version of this bug came back in October ([part 4](/blog/ebpf-build-log-4/)).
