---
title: "Learning eBPF by building a ransomware detector, part 6: back to the path problem"
date: 2026-11-25T10:06:00-05:00
weight: 6
series: "ebpf-build-log"
tags: ["ebpf", "linux", "vfs"]
description: "BpfJailer hit the same path resolution problem. What I'm working on now: walking the dentry tree to resolve full paths."
---

*November 2026, in progress.*

The path problem from [part 3](/blog/ebpf-build-log-3/) is still open, and it's what I'm working on now.

Watching the BpfJailer talk was reassuring here, because they ran into the same limitation. It comes from how the hooks are designed. The `inode_*` LSM hooks don't get the mount, so there is no `struct path` to resolve, and `bpf_d_path` can't help.

## The approach I'm taking

Instead of relying on `/proc/<pid>/cwd`, I'm walking the dentry tree in the kernel:

1. Starting from the dentry the hook gives me, follow the parent dentries up toward the root, reading each name.
2. Stop at a fixed depth, since the BPF verifier only accepts bounded loops.
3. Resolve the mount in user space, so the agent can place that chain of names in the real file system tree, including for containers.

The tradeoff is size. Sending a chain of names makes every rename and delete event bigger, and I spent part 4 learning how easy it is to overflow the ring buffer. So it has to be measured against the same baseline as everything else, with every dropped event counted.

I'll write up how it goes when it's done.
