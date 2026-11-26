---
title: "eBPF ransomware monitor, part 4: measuring honestly"
date: 2026-11-25T10:04:00-05:00
weight: 4
series: "ebpf-build-log"
tags: ["ebpf", "benchmarking", "edr"]
description: "Re-measuring the agent found 77% of events lost under load while it reported zero drops."
---

*October 25 to 28, 2026.*

When I came back to the project in late October, I asked Claude Code for an honest review of the whole thing, the kind an interviewer at an EDR company would give it, and compared the design with Meta's SysArmor (LPC 2023) and BpfJailer. The review found ways real ransomware would get past the detector, legitimate tools it would kill, pipeline bugs, and claims in my README that weren't true: "zero-allocation", "instant containment", "protecting the OS from kernel panics", "zero telemetry loss". I set one rule for my notes from then on: write down what is true, including what doesn't work.

## Measure before changing anything

The old "events per second" counted events entering the channel, not events analyzed. A single Python generator tops out at about 34,000 ops/s on its own, so the August "peak" was the generator's limit. The "15% CPU peak" was a lifetime average from `ps`; measured per interval it was 172%.

I rebuilt the harness (`run_baseline.py`) and recorded a proper baseline on October 25:

- All 12 simulation runs were contained, with 96.7 to 99.4% of files preserved and a median 1.7 to 70 ms from first malicious write to containment.
- Two false positives every time: `tar xf` of a tree with a `README.md` (the bare word "readme" matched the ransom-note list) and temp-file churn (write `.tmp`, rename, delete).
- With 4 parallel generators, 77% of events never reached the agent, while it reported 0 drops. The kernel ring buffer was overflowing and nothing counted it.

## Fixing what the baseline exposed

1. **Tests first.** Unit and scenario tests that replay each workload. Known false positives and misses are written as tests of the correct behavior and show as skipped until they're fixed.
2. **README fixes.** I checked every README claim against the code and measurements, removed overclaims like "instant containment", and added a Known Limitations section.
3. **Safe shutdown.** Ctrl+C now drains the queue, so it can't land between SIGSTOP and SIGKILL and leave a process frozen.
4. **Filter before reserve.** Noise events still took ring buffer space until the reader passed them. Filtering first kept 24,191 noise events off the buffer in one test.
5. **Stale bytes in events.** Ring buffer records aren't cleared between uses, so 114 of 163 sampled events carried leftover bytes past the terminator. Zeroing each record brought that to 0 of 24,473.
6. **Count every drop.** Failed ring buffer reservations are now counted, and each event's queue latency is measured from its kernel timestamp.
7. **Order per process.** Events now route by process to one of 32 workers, so a write is always handled before its delete, and scoring windows use kernel time. Queue latency p99 fell from about 11 s to about 1.2 s.

Every change after this was measured against the same baseline, with the old and new builds run side by side.
