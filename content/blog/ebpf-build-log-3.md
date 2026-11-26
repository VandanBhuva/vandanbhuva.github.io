---
title: "Learning eBPF by building a ransomware detector, part 3: the first detector"
date: 2026-09-08T18:00:00-04:00
weight: 3
series: "ebpf-build-log"
tags: ["ebpf", "go", "edr", "siem"]
description: "Scoring processes in Go, freezing and killing them, the race that let fast malware finish first, and the questions I left open."
---

*July 25 to early September, 2026.*

## Detection

The detection logic went in over three days:

1. Jul 25: absolute paths at `file_open` with `bpf_d_path`, and a Shannon entropy check on written files. Encrypted data looks close to random, so its entropy is near the maximum of 8 bits per byte.
2. Jul 26: filters for noisy files, write-then-delete correlation (write `report.docx.tmp`, delete `report.docx`, which is how a lot of ransomware replaces originals), ransom note detection, and a weighted threat score per process.
3. Jul 27: active response. When a score crosses 100 the agent freezes the process with `SIGSTOP`, takes a forensic snapshot from `/proc` (command line, executable, working directory, parent, open files), then kills it with `SIGKILL`. Freezing first means the process can't keep encrypting while I collect evidence.

The score had four parts, counted per process over a 5-second window:

- velocity: writes, renames and deletes in the window;
- indicators: ransom note names like `read_me.txt` and extensions like `.lockbit`;
- entropy of written files;
- write-then-delete correlation by file stem.

Developer tools like npm and cargo, and loggers, hit velocity hard and were getting killed just for being fast. So I capped velocity well below the kill threshold. Speed alone should never be enough; a process also had to do something that looks like encryption. I meant that cap to be 49. When I audited the code in October, it was actually 30, and the README said both. Entropy and correlation got caps too, so no single noisy signal can kill on its own.

## State in Go

Each process's state started as a Go map behind one global mutex. Under heavy I/O every event waited on that one lock. I split the map into 32 shards picked by PID, each with its own mutex, and replaced the growing slices of timestamps with fixed-size circular buffers so a busy process can't grow its own state without bound.

## The exclusion blind spot

To stop false positives on databases, I excluded high-entropy churn on `.sqlite` files. Then, testing a stealthier simulation, I found ransomware could write an encrypted `.sqlite.tmp` and delete the original `.sqlite`, and the delete was ignored, so correlation never fired. I tightened the exclusions so the delete still counted toward correlation. The lesson I took was that every exclusion is a gap someone can use. I learned it again in November, when an audit showed the exclusion lists still dropped every event for `.db` and `.sqlite` files ([part 8](/blog/ebpf-build-log-8/)).

## The kill that arrived too late

Testing against a fast Python encryptor, the agent printed `SIGSTOP failed: os: process already finished` again and again. The script finished its loop and exited in about 2.5 seconds, before the agent got to it. The reason was my single loop: read an event, read the file from disk to compute entropy (retrying up to five times, 10 ms apart, while the file was still empty), score it, log it. The kernel queued events instantly and user space fell further and further behind.

On August 15 I split reading from analysis. The main loop only pulls events off the ring buffer into a buffered Go channel of 10,000 events, and a pool of 8 worker goroutines does the file reads, entropy and scoring in parallel. The sharded map is what made that safe. The agent then stopped the same script mid-run. I later grew it to 32 workers and a 100,000-event channel.

That worker pool fixed one race and created another I didn't notice until October: with 32 workers on one shared channel, a process's delete could be scored before its write ([part 6](/blog/ebpf-build-log-6/)).

## Getting alerts somewhere useful

Every event, score change and containment step became one JSON line, and I shipped them into Datadog as a SIEM so I could see alerts the way a security team would. Datadog colors logs by standard syslog severities, and my MEDIUM and HIGH levels showed up as plain blue INFO. Rather than rename my levels to suit one product, I added a Datadog log pipeline with a category processor that maps `@level:MEDIUM` to WARN and `@level:HIGH` to ERROR, and a status remapper on top.

## Questions I left open

The last commits of this version went in at the start of September, when the semester started. I left myself a list of questions:

- Filter in the kernel or in user space? Filtering in the kernel saves the trip across, but string matching under the BPF verifier is painful, and my threat engine lives in user space where the kernel can't see it. There was also a difference in what each side can do: returning `-EPERM` from a hook blocks one operation, while a kill from user space stops the whole process.
- Entropy timing. The workers waited a fixed time for data to land in a file before measuring it. A process that waits out that window and then writes gets past it. My idea was to measure when the file is closed, from the `file_release` hook, or while the data is being written.
- Additive scoring. Scores just add up. A benign process doing many high-entropy writes would look like ransomware. Allowlists? Caps per signal?
- `path_rename` or `inode_rename`? That one gets its own post.

All four came back in October, and I'll point to where each was answered.
