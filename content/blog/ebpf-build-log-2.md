---
title: "Learning eBPF by building a ransomware detector, part 2: LSM hooks and the first detector"
date: 2026-11-25T10:02:00-05:00
weight: 2
series: "ebpf-build-log"
tags: ["ebpf", "linux", "lsm", "edr"]
description: "Why I moved from syscall tracepoints to BPF LSM hooks, and how the first scoring engine and kill path worked."
---

*July 24 to September 8, 2026.*

## Why tracepoints weren't enough

Tracepoints were fine for watching, but the more I read about how security tools use eBPF, the less they looked like the right base. Three things bothered me.

First, a syscall tracepoint sees the arguments the way the program passed them, including a pointer to a path in user memory. A malicious process can change that memory after my probe reads it and before the kernel uses it. That's a time-of-check to time-of-use (TOCTOU) race, and it means the path I logged might not be the file that was actually touched.

Second, syscalls aren't the only way into a file operation. `openat2` and io_uring reach the same kernel code without going through the syscall I was hooking.

Third, a tracepoint can only observe. If I ever wanted to block an operation instead of reacting to it, I'd need a hook that can return an error.

Kprobes on internal kernel functions solve some of this, but those functions have no stable interface and can change between kernel versions, which is a bad property for something meant to run on many machines.

So on July 24 I moved to BPF LSM hooks (`file_open`, `inode_rename`, `inode_unlink`). LSM hooks run inside the kernel's VFS after the kernel has copied the arguments in and resolved them to its own objects, at the point where it decides whether to allow the operation. That closes the user-memory race, catches every path into the operation, and leaves the option of denying it later. Tetragon makes the same choice. I also adopted CO-RE (`BPF_CORE_READ`), so one compiled binary can run across kernel versions with different struct layouts.

## The detector

The detection logic went in over three days:

1. **Jul 25:** absolute paths at `file_open` with `bpf_d_path`, and a Shannon entropy check on written files. Encrypted data looks close to random, so its entropy is near the maximum.
2. **Jul 26:** filters for noisy files, write-then-delete correlation (write `report.docx.tmp`, delete `report.docx`, which is how a lot of ransomware replaces originals), ransom note detection, and a weighted threat score per process.
3. **Jul 27:** active response. When a score crosses 100 the agent freezes the process with `SIGSTOP`, takes a forensic snapshot from `/proc` (command line, executable, working directory, parent, open files), then kills it with `SIGKILL`. Freezing first means the process can't keep encrypting while I collect evidence.

The score had four parts: velocity (writes, renames and deletes in the last few seconds), indicators (ransom note names like `read_me.txt` and extensions like `.lockbit`), entropy, and write-then-delete correlation. Fast developer tools and loggers hit velocity hard, so I capped it at 49. Being fast should never be enough to get a process killed; it had to do something that looks like encryption too.

Each process's state started as a Go map behind one global mutex. Under heavy I/O every event waited on that one lock, so I split the map into 32 shards picked by PID, each with its own mutex. I also swapped the growing slices of timestamps for fixed-size circular buffers, so a busy process can't make its own state grow without bound.

## Getting alerts somewhere useful

In August I made every event, score change and containment step one JSON line, and shipped them into Datadog (a free trial) as a SIEM so I could see alerts the way a security team would. Datadog colors logs by standard severities, and my MEDIUM and HIGH levels showed up as plain blue INFO. Rather than rename my levels to suit one product, I added a Datadog log pipeline that maps `@level:MEDIUM` to WARN and `@level:HIGH` to ERROR.

## The kill that arrived too late

Testing against a fast Python encryptor, the agent printed `SIGSTOP failed: os: process already finished` again and again. The script finished and exited before the agent got to it. The reason was my single loop: read an event, read the file to compute entropy (retrying up to five times, 10 ms apart, while the file was still empty), score it, log it. The kernel queued events instantly and user space fell further and further behind.

On August 15 I split reading from analysis. The main loop only pulls events off the ring buffer into a buffered Go channel, and a pool of worker goroutines does the file reads, entropy and scoring in parallel. The sharded map from July is what made that safe. The agent then stopped the same script mid-run. I later scaled to 32 workers and a 100,000-event buffer, ran four attack simulations, and tagged the result `baseline` in early September.

That worker pool fixed one race and created another I didn't notice until October: with 32 workers on one shared channel, a process's delete could be scored before its write.
