---
title: "Back to the path problem: walking the dentry tree"
date: 2026-11-25T18:00:00-05:00
weight: 10
series: "ebpf-build-log"
tags: ["ebpf", "linux", "vfs"]
description: "Why the path problem kept turning into other bugs, what BpfJailer's code showed me, and walking the dentry tree in the kernel."
---

With the first round of fixes measured, I'm back on the problem from [part 4](/blog/ebpf-build-log-4/): rename and unlink events only carry a file name, and the agent guesses the directory from the process's working directory.

## How it kept showing up

Over the last month the path problem turned up as three different bugs, each of which first looked like something else:

- The temp-file churn false positive. I blamed it on scoring. It was a stale write record that never moved to the file's new name, because the rename event had the wrong directory.
- The agent killing my desktop session. gnome-shell wrote a temp file and deleted it, and with the delete placed in the wrong directory, the delete matched a different write.
- Log lines naming the wrong directory, which matters to whoever reads the alert.

I fixed the first two in other ways: correlation now needs a write that was judged encrypted, and the desktop session is protected. The paths themselves are still wrong.

## What BpfJailer's code showed me

[BpfJailer is open source](https://github.com/facebookincubator/bpfjailer) now, and reading how it resolves paths answered most of what I asked in part 4. It also showed how much effort it takes. Two ideas carried over to my design.

First, the walk happens in the kernel, inside the hook. Rebuilding a path later, in user space, races with the file system changing underneath you. Walking the dentries at the moment the hook fires gives a consistent answer for the operation being checked.

Second, which view of the file system a path is reported in matters. Any user can create their own user and mount namespace and bind mount things wherever they want, which means the path a process sees can be made up. BpfJailer resolves paths as they look from systemd's mount namespace, the host's view, which an unprivileged user can't rearrange.

## The approach I'm taking

1. Start from the dentry the inode hook gives me and follow parent pointers toward the root, copying each name into the event along with its offset.
2. Stop at a fixed depth, since the verifier only accepts bounded loops, and mark the event if the path was cut short.
3. Place the result in the host's view of the mount tree, not in whatever the process's own namespace shows.

The full path won't fit on the 512-byte BPF stack, which I've been using as scratch space for the filters. This is where the per-CPU buffer I passed on in [part 6](/blog/ebpf-build-log-6/) comes back, with one slot per hook so two hooks on the same CPU can't overwrite each other.

The tradeoff is size. A chain of names makes every rename and delete event bigger, and I've learned how easy it is to overflow the ring buffer. So the change gets measured against the same baseline as everything else, with every dropped event counted. It should also give long-path events a real path, where they currently get a basename and a status flag.

After paths, the next step is moving more of the processing into the kernel. I'll write up both when they're done.

## Resources

- BpfJailer: eBPF based Mandatory Access Control, Linux Plumbers 2025: [talk page](https://lpc.events/event/19/contributions/2159/), [video](https://www.youtube.com/watch?v=o78vnXfih8E), [slides](https://lpc.events/event/19/contributions/2159/attachments/1833/3929/BpfJailer%20LPC%202025.pdf)
- SysArmor: Meta's eBPF Security Detection and Enforcement Tool, Linux Plumbers 2023: [talk page](https://lpc.events/event/17/contributions/1602/), [video](https://youtu.be/voQMOQ1a7QM?t=22892), [slides](https://lpc.events/event/17/contributions/1602/attachments/1231/2508/SysArmor%20LPC%202023.pdf)
- [BpfJailer source code](https://github.com/facebookincubator/bpfjailer)
