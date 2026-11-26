---
title: "Learning eBPF by building a ransomware detector, part 5: working through the checklist"
date: 2026-11-25T10:05:00-05:00
weight: 5
series: "ebpf-build-log"
tags: ["ebpf", "linux", "lsm", "edr"]
description: "Trusting process identity instead of names, and checking what was actually written to a file."
---

*October 30 to November 11, 2026.*

The next items on my checklist were about two assumptions v1 made: that I knew which process I was looking at, and that I knew what it had written.

## Which process is this, really?

### Names can be faked

My allowlist matched the process name (`comm`), which any process can change with `prctl(PR_SET_NAME)`. A test that named itself after a trusted tool got its activity skipped. Now each task's executable identity (device and inode) is recorded in BPF task storage at exec and inherited at fork. The agent resolves it once, hashes the binary with SHA-256, and only trusts root-owned, non-writable allowlisted binaries or listed hashes. Spoofed tests went from 0 to 49 of 50 files saved.

### Processes that started before the agent

Task storage only fills at exec and fork, so anything already running had no identity. A BPF task iterator now walks every running task once at startup (about 850 on my VM) and fills it in.

### PIDs get reused

Containment was keyed by PID alone, and Linux reuses PIDs. In a test that cycles the PID counter, the old agent punished an innocent process that inherited a dead process's PID. Now a process is its PID plus start time, and signals go through a pidfd checked against that start time, so a kill can only reach the process that was scored.

### One rename shouldn't be a kill

A single `mv backup.tar backup.tar.enc` scored 100. Indicators are now counted over a 30-second window: several renames to a known extension, or new notes appearing in three or more directories.

### It killed my own desktop

During live tests the agent killed `gnome-shell` (its temporary image files looked like write-then-delete) and the host's `logrotate`. After that, the desktop session went on the protected list.

## What was actually written?

v1 checked a file's entropy when it was opened for writing, which is before the new bytes exist. An in-place overwrite was read as its old plaintext, and memory-mapped writes produced no event at all.

I added three hooks: `file_permission` for writes (it covers `write`, `pwrite`, `writev`, `splice` and io_uring), `mmap_file` for shared writable mappings, and `file_release` for the final close. The agent reads a file's "before" state at open and its "after" state at close, then asks three questions in order. Did the format break, like a PDF or zip losing its header, or text that stopped being text? Did entropy rise at the same offset? Is the content close to uniform random by entropy and a chi-square test? Compressors keep valid headers, so `gzip`, `xz` and `zstd` pass the first check and don't get flagged. Requiring an encrypted-looking write to also break the format is what keeps compressors out.

I also removed blanket skips for database files and paths like `/snap/`, because an attacker picks the file name, and made rename-over-a-file and link-then-delete correlate through the inode.

## Results

Re-running the October baseline against both builds: no loss in files preserved (outbreak median 985 vs 983.5 of 1,000), false positives down from 6 of 12 runs to 0 of 24, and every lost event counted, where the baseline lost up to 64% silently. I tagged this point `phase1-done` on November 11.
