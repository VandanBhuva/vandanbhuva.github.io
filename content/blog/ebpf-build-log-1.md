---
title: "Learning eBPF by building a ransomware detector, part 1: from tutorials to first hooks"
date: 2026-11-25T10:01:00-05:00
weight: 1
series: "ebpf-build-log"
tags: ["ebpf", "linux", "edr"]
description: "How I learned eBPF from the eunomia tutorials and got file events out of the kernel into a Go agent."
---

This series is how I learned eBPF by building the [eBPF Ransomware Behavior Monitor](/projects/ebpf-ransomware-monitor/), a Linux agent that spots ransomware from what it does to files and stops it mid-attack. I've written it in the order things happened, including the parts where my first design turned out to be wrong.

| Part | Dates | What happened |
| --- | --- | --- |
| 1. Tutorials to first hooks | early Jul to Jul 20 | Learned eBPF, got file events into a Go agent |
| 2. LSM hooks and the first detector | Jul 24 to Sep 8 | Moved to LSM, built scoring and the kill path |
| 3. The path problem | Jul to Aug | Rename and unlink hooks couldn't give me full paths |
| 4. Seeing how Meta does it | Oct 25 to 28 | SysArmor and BpfJailer showed me what my design was missing |
| 5. Working through the checklist | Oct 30 to Nov 11 | Process identity, content checks, false positives 6/12 to 0/24 |
| 6. Back to paths | now | Walking the dentry tree to resolve full paths |

## Learning eBPF first

I started in early July with no eBPF experience. I worked through the eunomia eBPF tutorials to learn the basics: what an eBPF program is, how it gets loaded and verified, how maps and ring buffers move data between the kernel and user space, and where programs can attach. I'd already used it a little for a small [network threat detector](/projects/network-threat-detector/), and I wanted a bigger security project that went into the file system instead. Ransomware detection fit: ransomware has a very recognizable pattern of file activity, and seeing file activity is exactly what eBPF is good at.

## The plumbing

Before any detection logic I needed a working toolchain: a `vmlinux.h` generated from the running kernel's BTF so my C code could use kernel types, and a repeatable way to compile and load a BPF program.

For hooks, I started with syscall tracepoints. They were the easiest way to see file activity while I was still learning.

- **Jul 18:** tracepoints on `sys_enter_openat`, then `sys_enter_renameat2` to see files being modified.
- **Jul 19:** a BPF ring buffer to carry events to user space. A ring buffer keeps events in order and lets the kernel side reserve space for an event before filling it in.
- **Jul 20:** a Go agent using `bpf2go`, which generates Go bindings for the BPF objects so the agent can load the program and read the ring buffer without hand-written glue.

## First bug

The first real bug was in string handling. Filenames come across as fixed-size C arrays, and Go printed everything in the array, including whatever bytes sat after the null terminator. Parsing the strings up to the terminator fixed it. A related bug came back in October, when I found the kernel side was sending stale bytes too ([part 4](/blog/ebpf-build-log-4/)).
