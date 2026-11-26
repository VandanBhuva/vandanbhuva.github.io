---
title: "eBPF ransomware monitor, part 3: making it usable and fast"
date: 2026-11-25T10:03:00-05:00
weight: 3
series: "ebpf-build-log"
tags: ["ebpf", "go", "datadog", "edr"]
description: "JSON telemetry into Datadog, and a race where the malware finished before the kill arrived."
---

*August 13 to September 8, 2026.*

## Telemetry into Datadog

In mid-August I worked on making the output useful to a security team. Every event, score change and containment step became one JSON line on stdout, shipped into Datadog (a free trial) as a SIEM, mostly so I could see the alerts visually. The Datadog agent's own telemetry writes then showed up as noise, so I added path exclusions for it.

Datadog colors log lines by standard severities (INFO, WARN, ERROR, CRITICAL). My levels were MEDIUM and HIGH, which it didn't recognize, so every alert showed up blue as INFO. Instead of renaming my levels to suit one SIEM, I added a Datadog log pipeline: a category processor mapping `@level:MEDIUM` to WARN and `@level:HIGH` to ERROR, then a status remapper on the result. After that the alerts came through with the right colors.

## The kill that arrived too late

The worst bug of v1 showed up in the simulations. Against a fast Python encryptor, the agent printed line after line of `SIGSTOP failed: os: process already finished`. The script finished its loop and exited while the agent was still working through its events. One loop did everything: read an event, read the written file to compute its entropy (retrying up to five times, 10 ms apart, while the file was still empty), score it, log it. The kernel queued events instantly and user space fell behind, so the kill arrived after the process was gone.

On August 15 I split ingestion from analysis. The main loop now only pulls events off the ring buffer and pushes them into a buffered Go channel (10,000 events), and a pool of 8 worker goroutines does the file reads, entropy and scoring. The sharded map from July let the workers update scores in parallel. With that change the agent stopped the same script mid-run. On August 19 I made ingestion non-blocking and scaled to 32 workers and a 100,000-event buffer.

The pool fixed the race but cost something I didn't see until October: with 32 workers on one shared channel, a process's delete could be handled before its write. [Part 4](/blog/ebpf-build-log-4/) covers that.

- **Aug 14:** split `main.go` into modules and added a Makefile.
- **Aug 15:** four simulations (benign I/O, ransom note plus `.lockbit` renames, stealthy `.tmp` write and delete, in-place outbreak), each with a write-up and Datadog screenshots.
- **Aug 19:** capped each scoring signal after benign I/O scored too high, and added benchmarks: 99% of 1,000 files preserved, about 35,000 events/s with a reported 0% drop rate, about 15 MB idle.
- **Sep 3 to 8:** an architecture diagram and scoring details in the README. This commit became the `baseline` tag.

## Questions I left open after v1

When I stopped in August I had four design questions written down. Two were settled later, and two are still partly open.

1. **`path_rename` or `inode_rename`?** The path hooks carry a `struct path`, which in principle gives `bpf_d_path` the mount it needs, while the inode hooks sit deeper and see only dentries. Walking the dentry chain in the kernel and sending each piece to user space works, but it multiplies the data per event. This is still open. The current plan is to walk the parent dentries in the kernel up to a fixed depth and resolve mounts in user space.
2. **Filter in the kernel or in user space?** Heavy filtering in BPF loads the kernel path and fights the verifier on string matching, and my scoring engine lives in user space where the kernel can't see it. On the other hand, returning EPERM from a hook stops one operation immediately, while a user-space kill stops the whole process later. In October I moved the cheap filters ahead of the ring buffer reservation (part 4).
3. **The entropy check could be timed around.** Workers read the file after a short wait, so a process that waited longer before writing would be read while the file was still old or empty. I wanted to compute it at write time or when the file is closed. The November rewrite ([part 6](/blog/ebpf-build-log-6/)) did the second: it checks the file when `file_release` fires.
4. **Additive scoring and false positives.** Scores just added up, so a benign program doing several high-entropy writes could get killed. Caps were my first answer: on August 19 I capped entropy at 60 and correlation at 80, and lowered velocity to 30, so no single signal can reach 100. Since November an encrypted write also has to pass a format check, which is what keeps compressors out.
