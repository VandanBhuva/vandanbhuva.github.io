---
title: "Learning eBPF by building a ransomware detector, part 4: seeing how Meta does it"
date: 2026-11-25T10:04:00-05:00
weight: 4
series: "ebpf-build-log"
tags: ["ebpf", "benchmarking", "edr"]
description: "Meta's SysArmor and BpfJailer talks showed me what my design was missing, so I measured properly and made a checklist."
---

*October 25 to 28, 2026.*

I paused the project in August for the fall semester. When I came back in late October, I wanted to know how people who run eBPF security at scale design these systems, so I watched Meta's talks on SysArmor (LPC 2023) and BpfJailer.

## What the talks showed me

Comparing their designs with mine made it clear that v1 had a lot of issues. SysArmor does much more of its filtering and scoring in the kernel, while I shipped nearly everything to user space and filtered there. Thinking like an attacker against my own agent, I could see ways real ransomware would get past it, legitimate tools it would kill, and bugs in the pipeline.

I also went back over my own README and found claims I couldn't back up: "zero-allocation", "instant containment", "protecting the OS from kernel panics", "zero telemetry loss". I set a rule for my notes from then on: write down what is true, including what doesn't work.

## Measuring before changing anything

My August numbers turned out to measure the wrong things. "Events per second" counted events entering the channel, not events analyzed, and a single Python generator tops out around 34,000 ops/s on its own, so the "peak" was really the generator's limit. The "15% CPU peak" was a lifetime average from `ps`; per interval it was 172%.

I rebuilt the test harness and recorded a real baseline on October 25:

- All 12 simulation runs were contained, with 96.7 to 99.4% of files preserved and a median 1.7 to 70 ms from the first malicious write to containment.
- Two false positives every time: `tar xf` of a tree containing `README.md` (the bare word "readme" matched my ransom-note list), and normal temp-file churn (write `.tmp`, rename, delete).
- With 4 parallel generators, 77% of events never reached the agent, and it reported 0 drops. The kernel ring buffer was overflowing and nothing counted it.

That last result changed how I thought about the project: a detector that silently misses three quarters of its events under load can't be trusted, however good its scoring is.

## The checklist

From there I wrote a checklist of changes and have been working through it since. The first fixes were the ones the baseline exposed:

1. Tests. Scenario tests that replay each workload. Known false positives and misses are written as tests of the correct behavior and stay skipped until fixed, so the gaps are visible.
2. An honest README, with a Known Limitations section.
3. Safe shutdown. Ctrl+C drains the queue, so it can't land between `SIGSTOP` and `SIGKILL` and leave a process frozen.
4. Filter before reserving. Noise events were taking ring buffer space before being thrown away. Filtering in the kernel first kept 24,191 noise events off the buffer in one test.
5. Stale bytes. Ring buffer records aren't cleared between uses, so 114 of 163 sampled events carried leftover bytes after the terminator, the kernel-side cousin of my first bug. Zeroing each record took that to 0 of 24,473.
6. Count every drop, and measure each event's queue latency from its kernel timestamp.
7. Order per process. Events now go to one of 32 workers based on the process, so a write is always handled before its delete. Queue latency p99 fell from about 11 s to about 1.2 s.

From then on, every change was measured against the same baseline, with the old and new builds run side by side.
