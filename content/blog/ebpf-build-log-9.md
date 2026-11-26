---
title: "Learning eBPF by building a ransomware detector, part 9: measuring phase one"
date: 2026-11-25T10:09:00-05:00
weight: 9
series: "ebpf-build-log"
tags: ["ebpf", "benchmarking", "edr"]
description: "Re-running the October baseline against both builds on the same day, what improved, what got worse, and what's still open."
---

*November 11, 2026.*

At the end of the first round of fixes I re-ran the October benchmark. I didn't compare against the October numbers alone, because the kernel had been updated and the VM now ran more background software, so the load generator itself ran faster than it did then. Instead I built both versions and ran them on fresh agents the same day. I tagged the result `phase1-done`.

## What improved

- Detection held. All four attack simulations were contained on every run of both builds. On the 1,000-file outbreak, 10 runs each: baseline median 985 files preserved (9.4 ms to containment), new build 983.5 (9.9 ms).
- False positives went from 6 of 12 runs to 0 of 24. `tar` with a README, temp-file churn, gzip and log rotation all stay at or below 30.
- Every lost event is counted. The baseline lost up to 64% of events in this run with nothing counting them. The new build tells you exactly where events were lost, in the kernel or at a worker's queue.
- Events are fresh. The backlog that took 17.8 seconds to drain on the baseline is gone (1.0 s), and the longest wait was 1.4 seconds.
- Every live check from the earlier parts passed again on the final build: spoofed names, pre-started processes, PID reuse, indicators, content checks, exclusions and rename-over-file.

I also left a fresh agent running for 20 minutes on the idle desktop. Nothing outside the tests went above velocity 24.

## What got worse

The new build sees 5 events per written file instead of 3 and has a 4x larger ring buffer, so the comparison isn't purely about code, and I noted that next to the numbers.

Per-process throughput is lower. One process is one worker, about 3,500 events per second, so a single very busy writer now gets only 17% of its events analyzed (the baseline managed 80%, by queueing them for seconds instead). The load generator also runs about 30% slower with the new agent loaded. Memory sits at 80 to 92 MB after the detection runs, against 39 to 71 MB before, and isn't returned.

## What's still open

Write-then-delete correlation still pairs files by name. Ransomware that gives its output random names, or renames the original away before deleting it, stays at 90 and isn't contained. I showed this with a live test on both builds and listed it in Known Limitations. The fix I want is to pair by data flow instead: the process read the file it's deleting.

The bigger ideas from BpfJailer are still ahead: denying writes in the kernel for a process that's already scored HIGH instead of waiting to kill it, and protecting the agent itself. Right now, anyone with root can stop it.

And the path problem from August was still there. Rename and unlink events still carried basenames joined to the process's working directory. It caused the churn false positive, the gnome-shell kill, and log lines that named the wrong directory. That's what I'm working on now.
