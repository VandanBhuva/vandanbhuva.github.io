---
title: "Learning eBPF by building a ransomware detector, part 6: fixing the pipeline"
date: 2026-11-25T10:06:00-05:00
weight: 6
series: "ebpf-build-log"
tags: ["ebpf", "ring-buffer", "go", "concurrency"]
description: "Filtering before reserving ring buffer space, stale bytes in records, counting every drop, and keeping each process's events in order."
---

*October 27 to 28, 2026.*

The baseline showed the pipeline between the kernel and my scoring code was losing events, reordering them, and timing them wrong. These four fixes came first, each measured against the baseline.

## Filter before reserving

Every hook reserved a 544-byte ring buffer record first, ran the noise filter afterwards, and discarded the record if it matched. I'd assumed a discarded record frees its space. It doesn't: it stays in the ring until the consumer reads past it. So noise (writes to `/dev` and `/proc`, browser IPC files) took the same space as real events and pushed the buffer toward overflow. Reserving also takes a spinlock shared by every CPU.

Moving the filter in front of reserve meant the filter needed the path somewhere else first. The obvious place is a per-CPU array used as scratch space. I didn't use it, because on this kernel BPF LSM programs run with migration disabled but can still be preempted, so two hooks on the same CPU could overwrite each other's buffer mid-use. The program stack is private to each run and needs no lookup, and 256 bytes fits in the 512-byte BPF stack. For rename, reading both names onto the stack would go over the limit, so I only read an 8-byte prefix, which is all the filter checks. While checking the generated code I also found that the buffer's alignment made a 4x difference in program size.

In a noise test, 24,191 events were filtered before reserve and never touched the ring buffer. Detection didn't change.

## Stale bytes and failed paths

Ring buffer memory isn't cleared between records, and my hooks only wrote some fields. To see what that meant, I made a throwaway build that inspected the raw bytes of each record instead of trusting the parsed strings. 114 of 163 events had leftover bytes after a string's terminator. The cause turned out to be `bpf_d_path` itself: it builds the path at the end of the buffer and moves it to the start, so the tail still holds a piece of the same path, and my 256-byte copy carried it into the record. I now zero each record and copy strings with `bpf_probe_read_kernel_str`, which stops at the terminator. After that, 0 of 24,473.

The same run found a filter that had never worked. I was matching `anon_inode` opens as if the path started with `/`, but `d_path` gives `anon_inode:[pidfd]` with no slash, so it never matched and those events were using ring space for nothing.

I was also ignoring `bpf_d_path`'s return value. A path longer than the buffer produced an event whose filename was leftover bytes. Dropping those events would hand anyone a free way to hide: just work in a deeply nested directory. Instead the event now carries a status field and the file's basename, which always fits and still drives velocity and the ransom note check. I made room for the field by narrowing another one, so the event didn't grow.

## Count every drop

A failed `bpf_ringbuf_reserve` just returned, which is how 77% of events vanished with 0 drops reported. I added a slot for it in a per-CPU stats map that Go already reads.

I also wanted to know how old an event was when a worker picked it up. `bpf_ktime_get_ns` is `CLOCK_MONOTONIC`, and Go can read the same clock directly, so latency is a subtraction on one clock with no wall-clock conversion. I record it in a fixed histogram of atomic counters, so recording is one atomic add and nothing allocates.

With counting in place, every generated event was either analyzed or counted as lost. The counters also showed something I didn't expect: the Go queue was never the problem. It peaked at 31,452 of 100,000 and dropped nothing. The ring buffer overflowed because one goroutine reads it, and 256 KB holds only about 470 events, roughly 10 ms at that rate. Queue latency was worse than I thought, too. Events waited up to 11 seconds, twice my 5-second scoring window.

## Keep each process's events in order

Two problems came from the worker pool I added in August.

First, order. 32 workers read one shared channel, so one worker could still be computing a write's entropy while another took the delete that followed it. The delete found no write, and the write-then-delete correlation, which is the signal for "encrypt a copy and delete the original", was missed.

Second, time. Every score used `time.Now()` at processing time. After 10 seconds in a queue, a burst could be spread out and scored as slow, or events far apart could be scored as a burst.

Now each event goes to a fixed worker chosen by `pid % 32`. The PID in the event is the thread group ID, so all threads of a process land on the same worker. Order then comes from the channel's FIFO, each worker owns its processes' state, and the 32 shard mutexes from July are gone. Scores use the kernel timestamp. The single 100,000-slot queue became 32 queues of 3,125, since 100,000 per worker would have been 1.7 GB.

The first live run of this caught a regression I'd created. The entropy check slept 10 ms between retries while waiting for a short file to fill up. With one worker per process, that sleep became something an attacker could use: write 50 small files first, and the worker spends about 2.7 seconds sleeping before it reaches the real encryption. Time to containment went from 60 ms to 2,740 ms. I replaced the sleep with a timer that hands the file back to its worker when it's time to read again, so the worker never sleeps. Containment came back to about 6 ms.

After this, nothing waited more than about 1.2 seconds in a queue, down from 11. The tradeoff is that one very busy process is now capped at one worker's throughput (about 3,800 events per second), and its extra events get dropped instead of waiting. They're counted, but it's on the list.
