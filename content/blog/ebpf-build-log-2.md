---
title: "eBPF ransomware monitor, part 2: moving to LSM hooks and building the detector"
date: 2026-11-25T10:02:00-05:00
weight: 2
series: "ebpf-build-log"
tags: ["ebpf", "linux", "lsm", "edr"]
description: "Why syscall tracepoints are the wrong base for a security tool, and the first scoring engine."
---

*July 24 to 27, 2026.*

## Why tracepoints weren't enough

Tracepoints were fine for watching, but the more I read, the less they looked like the right base for a security tool. A tracepoint on a syscall sees the arguments as the program passed them, including a pointer to a path in user memory. A malicious process can change that memory after the probe reads it and before the kernel uses it, a time-of-check to time-of-use (TOCTOU) race. Syscall hooks also miss other ways into the same operation, such as `openat2` and io_uring, and they can only observe, never refuse. Kprobes on internal kernel functions can see more, but those functions have no stable interface and can change between kernel versions.

So on July 24 I moved to BPF LSM hooks (`file_open`, `inode_rename`, `inode_unlink`) and adopted CO-RE (`BPF_CORE_READ`), so one binary runs across kernel versions. LSM hooks run inside the VFS, after the kernel has copied the arguments in and resolved them to its own objects, at the point where it decides whether to allow the operation. That closes the user-memory race, covers every path into the operation, and leaves the option of denying it in the kernel later. Tetragon makes the same choice.

## The path problem

The move created a new problem: paths. At `file_open` I could build the absolute path with `bpf_d_path`. The rename and unlink hooks work on dentries without the mount, so `bpf_d_path` can't be used there and all I had was the file's basename. My workaround was to send the basename from the kernel and have the Go agent join it to the process's working directory from `/proc/<pid>/cwd`.

## The detection engine

The detection engine went in over three days:

1. **Jul 25:** absolute paths through `bpf_d_path`, and a Shannon entropy check on written files to spot encryption.
2. **Jul 26:** filename filters for noise, write-then-delete correlation (write `report.docx.tmp`, delete `report.docx`), ransom note detection, and a weighted threat score per process kept in fixed-size circular buffers.
3. **Jul 27:** active mitigation. When a score crosses 100 the agent freezes the process with `SIGSTOP`, takes a forensic snapshot from `/proc` (command line, executable, working directory, parent, open files) and then kills it with `SIGKILL`.

The score had four parts: velocity (how many writes, renames and deletes in the last few seconds), indicators (ransom note names like `read_me.txt` and known extensions like `.lockbit`), Shannon entropy of written files, and write-then-delete correlation. Fast developer tools and loggers hit velocity hard, so I capped it at 49. Speed alone could never reach the kill threshold of 100; a process also needed one of the other signals.

I kept each process's state in a Go map behind one global mutex at first. Under heavy I/O every event waited on that lock, so on July 26 I split the map into 32 shards, picked by PID, each with its own mutex. I also replaced the growing slices of timestamps with fixed-size circular buffers, so each process's state has a fixed size.

To avoid false positives I also excluded database files (`.sqlite`, `.db` and their journals) from the checks. Those blanket skips stayed in the code until the November exclusion audit ([part 6](/blog/ebpf-build-log-6/)) removed them, since an attacker picks the file name.

By the end of July the monitor could watch file activity, score it and stop a process mid-attack.
