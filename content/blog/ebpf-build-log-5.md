---
title: "eBPF ransomware monitor, part 5: who is this process, really?"
date: 2026-11-25T10:05:00-05:00
weight: 5
series: "ebpf-build-log"
tags: ["ebpf", "linux", "edr"]
description: "Fixing process name spoofing, processes started before the agent, and PID reuse."
---

*October 30 to November 9, 2026.*

This round went after the assumptions the agent made about which process it was looking at.

## Process names can be faked

The allowlist and the list of protected processes matched `comm`, which any process can set with `prctl(PR_SET_NAME)`. A simulation calling itself `node` had its opens skipped (0 of 50 files saved), and one calling itself `sshd` was refused containment 51 times.

The fix keeps each task's executable identity (device and inode) in BPF task storage, set at exec and inherited at fork. The agent resolves it once through `/proc/<pid>/exe`, hashes it with SHA-256, and only trusts a root-owned, non-writable allowlisted binary or a listed hash. All three spoofs went from 0 to 49 of 50 files saved.

## Processes that started first

Task storage only fills at exec and fork, so malware already running before the agent had no kernel identity. An `iter/task` program now walks all (about 850) running tasks once at startup and seeds their identity.

## PIDs get reused

Containment was keyed by PID alone. Using `test/reuse_pid.py`, which wraps the PID counter without root, the old agent killed an innocent process that inherited a dead process's score, and ignored ransomware that reused a contained PID. Now a process is its PID plus start time, and signals go through a pidfd checked against that start time.

## Sticky indicators

A single `mv backup.tar backup.tar.enc` scored +100 and was a kill. Both indicators are now counted over a 30-second window: three renames to a known extension, or notes newly created in three or more directories backed by a rename or encrypted write.

I also split the overgrown `main.go` by topic without changing any code.

## It killed my own desktop

During live tests on November 9, the agent scored `gnome-shell` at 80 for correlation plus velocity from its temporary image files and killed it, which ended the desktop session. I thought the VM had restarted. A day earlier, while I was testing PID reuse, it had also killed the host's own `logrotate` at a score of 110. Both cases led to fixes in [part 6](/blog/ebpf-build-log-6/).
