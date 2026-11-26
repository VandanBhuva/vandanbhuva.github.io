---
title: "Learning eBPF by building a ransomware detector, part 1: from tutorials to first hooks"
date: 2026-11-25T10:01:00-05:00
weight: 1
series: "ebpf-build-log"
tags: ["ebpf", "linux", "edr"]
description: "How I learned eBPF from the eunomia tutorials and got file events out of the kernel into a Go agent."
---

This series is how I learned eBPF by building the [eBPF Ransomware Behavior Monitor](/projects/ebpf-ransomware-monitor/), a Linux agent that spots ransomware from what it does to files and stops it mid-attack. It's in the order things happened. The first four parts cover the version I built over the summer. The rest cover what I changed after watching two of Meta's talks on eBPF security in October and realizing how much of that first version was wrong.

| Part | Dates | What happened |
| --- | --- | --- |
| 1. Tutorials to first hooks | Jul | Learned eBPF, got file events into a Go agent |
| 2. Moving to LSM hooks | Jul 24 | Why syscall tracepoints weren't safe enough |
| 3. The first detector | Jul 25 to Sep | Scoring, the kill path, a race, and a SIEM |
| 4. The path problem | Jul to Aug | Rename and unlink hooks couldn't give me full paths |
| 5. Seeing how Meta does it | Oct 25 to 26 | SysArmor and BpfJailer, a hard review, an honest baseline |
| 6. Fixing the pipeline | Oct 27 to 28 | Drops, stale bytes, ordering, queue latency |
| 7. Which process is this? | Oct 30 to Nov 9 | Executable identity, bootstrapping, PID reuse |
| 8. What was actually written? | Nov 9 to 11 | Content checks, exclusions, renames over files |
| 9. Measuring phase one | Nov 11 | Re-running the baseline, and what's still open |
| 10. Back to paths | now | Walking the dentry tree to resolve full paths |

## Learning eBPF first

I started in early July with no eBPF experience and worked through the eunomia eBPF tutorials: what an eBPF program is, how the kernel loads and verifies it, how maps and ring buffers move data between the kernel and user space, and where programs can attach. I'd used it a little for a small [network threat detector](/projects/network-threat-detector/) and wanted a bigger security project that went into the file system. Ransomware fit. It has a very recognizable pattern of file activity, and watching file activity is what eBPF is good at.

## The plumbing

Before any detection logic I needed a toolchain: a `vmlinux.h` generated from the running kernel's BTF so my C code could use kernel types, and a repeatable way to compile and load a BPF program.

For hooks I started with syscall tracepoints, the easiest way to see file activity while I was still learning.

- Jul 18: tracepoints on `sys_enter_openat`, then `sys_enter_renameat2`, to see files being modified.
- Jul 19: a BPF ring buffer to carry events to user space. It's shared across CPUs, keeps events in order, and lets the kernel side reserve space for a record and fill it in place instead of copying it.
- Jul 20: a Go agent using `bpf2go`, which generates Go bindings for the BPF objects so the agent can load the program and read the ring buffer without hand-written glue.

## First bug

The first real bug was string handling. Filenames come across as fixed-size C arrays, and Go printed the whole array, including whatever bytes sat after the null terminator. Parsing up to the terminator fixed it. A cousin of this bug came back in October on the kernel side, where the records themselves were carrying stale bytes ([part 6](/blog/ebpf-build-log-6/)).
