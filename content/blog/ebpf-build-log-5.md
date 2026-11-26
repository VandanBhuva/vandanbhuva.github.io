---
title: "Learning eBPF by building a ransomware detector, part 5: seeing how Meta does it"
date: 2026-10-26T18:00:00-04:00
weight: 5
series: "ebpf-build-log"
tags: ["ebpf", "edr", "testing", "benchmarking"]
description: "What Meta's SysArmor and BpfJailer talks showed me about my design, a hard review of v1, and an honest baseline before changing anything."
---

*October 25 to 26, 2026.*

I paused the project in August for the fall semester. When I came back in late October I wanted to know how people who run eBPF security at scale design these systems, so I watched two talks from Meta at Linux Plumbers: SysArmor (LPC 2023) and BpfJailer (LPC 2025, by Liam Wisehart).

## SysArmor

What I took from SysArmor was mostly about the kernel side, and most of it turned into items on my checklist:

- filter events before reserving ring buffer space for them;
- count every dropped event;
- find out about processes that were already running when the agent started, with a BPF iterator;
- build paths by walking dentries in the kernel;
- dedupe repeated events the first time they're seen, in the kernel;
- time each hook.

My agent did almost none of this. It reserved space for every event and filtered afterwards, didn't count drops, and only knew about processes that did something after it started.

## BpfJailer

BpfJailer is Meta's eBPF LSM based mandatory access control and jailing system. They use it to sandbox untrusted code, restrict access to sensitive data, and protect confidential-computing processes even from root. It tracks processes with BPF task storage that gets copied to children on `clone()`, and when a jailed process tries something it isn't allowed to do, the LSM hook denies it on the spot.

Putting it next to mine was uncomfortable:

| | My monitor | BpfJailer |
| --- | --- | --- |
| Model | Detection: observe, score, react afterwards | Prevention: define what a jailed process may do |
| LSM return value | Always 0 | Denies, synchronously |
| Process identity | PID in user-space maps | Task storage inherited on clone |
| Response | Signals from user space, after a delay | The operation never happens |
| Resisting root | None | Yes |

My LSM hooks always returned 0, so I was using the most security-relevant hook type in the kernel as a tracepoint. The ideas I took were task storage with inheritance for process identity, denying in the kernel, protecting the agent itself, and a graduated response where HIGH means "jail" (writes fail, the process lives) and only CRITICAL means kill. That last one makes a false positive recoverable.

There's a real difference I didn't want to paper over, though. BpfJailer applies a known policy to processes chosen in advance. My detector has to judge processes it knows nothing about, which is harder and will always produce more false positives.

The talk also mentioned they had run into the same path resolution problem I'd been stuck on since August ([part 4](/blog/ebpf-build-log-4/)), which made me feel better about it.

## Reviewing v1 like an attacker

With all that in mind I went through my own agent the way someone trying to get past it would. The list was long. Real ransomware could avoid it by encrypting files in place, by replacing originals with a rename instead of a delete, by giving itself a trusted process name, by working in file types or directories I'd excluded, or by flooding or stalling the agent so its own events got lost. On the other side, it would kill legitimate tools: `tar` and package installs (anything that wrote a file called README), `gzip` and logrotate, and normal temp-file churn.

There were pipeline bugs too. Per-process event order was lost across workers. The kernel's event timestamp was captured but never used. Containment was keyed by PID, which Linux reuses. Kernel drops were invisible.

Then I read my README again. It claimed "zero-allocation", "instant containment", "zero telemetry loss", and "protecting the OS from kernel panics" (a full ring buffer just makes reserve fail; nothing panics). I couldn't back most of it.

## An honest baseline

Before changing anything I tagged the code as it was (`baseline`) and checked it built from a clean clone, so every number afterwards is a diff against a known commit.

My August benchmarks turned out to measure the wrong things. "Events per second" counted events entering the queue, not events analyzed. With metrics enabled, the stress directory was excluded from analysis, so the expensive part never ran. The ~35,000 "peak" was the speed of the Python load generator itself, which does about 34,000 ops/s with no agent running. The "15% CPU peak" was a lifetime average from `ps`; measured per interval it peaked at 172%.

I rebuilt the harness with a dry-run mode that logs "would contain" instead of signalling, so throughput can be measured without the agent killing its own load generator, and measured detection separately with containment on. On October 25:

- Detection on my own simulations: 12 of 12 runs contained, 96.7 to 99.4% of files preserved, median 1.7 to 70 ms to containment.
- False positives: `tar` extraction with a README 3 of 3, temp-file churn 3 of 3, gzip of 8 files 1 of 1.
- With 4 parallel generators, 77% of events never reached the agent, and it reported 0 drops.
- With 1 generator nothing was lost, but the queue reached 55,677 events and took 20 seconds to clear.

The 77% changed how I saw the whole project. A detector that silently misses three quarters of its events under load can't be trusted, however good its scoring is.

## The checklist, and the first items

I turned the review into a checklist and have been working through it since. The first few items were about being able to measure anything at all.

Tests. There were none. I turned every false positive and evasion from the review into a scenario test that goes through the same `processEvent` path as live events. Each one asserts the correct behavior and names the checklist item that fixes it. Until then it shows up as a skip; once it starts passing, the test fails until I remove the marker. That's the same idea as pytest's `xfail(strict=True)`: the suite stays green, so it works as a regression gate, but known bugs stay visible. Replaying the benchmark workloads in the tests reproduced the measured scores exactly (gzip of 8 files: 104), and it also showed me one of my diagnoses was wrong. I'd blamed the temp-file churn false positive on scoring. It was actually the path bug from part 4: the rename event only had a basename, the agent guessed the wrong directory, and a stale write record then matched the delete.

The README. I rewrote it so every number has a measurement behind it, and restored a Known Limitations section that lists how the detector can be evaded. The audit also turned up details I'd misremembered: the velocity cap was 30, not 49, and one demo I'd described as caught by entropy had entropy 0 at the kill. It was caught only by the `.enc` extension.

Shutdown. Making Ctrl+C drain the queue looked like one WaitGroup. But containment ran in its own goroutine, so a Ctrl+C at the wrong moment could leave a process frozen by `SIGSTOP` forever with no forensic record. Shutdown now waits for those too. I also replaced a log line per dropped event (extra work exactly when the agent is overloaded) with a once-a-second summary.

The next items were the pipeline itself.
