---
title: "Learning eBPF by building a ransomware detector, part 10: back to the path problem"
date: 2026-11-25T18:00:00-05:00
weight: 10
series: "ebpf-build-log"
tags: ["ebpf", "linux", "vfs"]
description: "Why switching hooks wouldn't have fixed full paths, and walking the dentry tree in the kernel instead."
---

*November 2026, in progress.*

The path problem from [part 4](/blog/ebpf-build-log-4/) is what I'm working on now. By this point it had shown up as three different bugs that each looked like something else: a false positive I blamed on scoring, the agent killing my desktop session, and log lines naming the wrong directory.

## Why switching hooks isn't the answer

In August my main idea was to move from `inode_rename` and `inode_unlink` to `path_rename` and `path_unlink` and call `bpf_d_path`. Looking at it again, `bpf_d_path` only works from a fixed allowlist of hooks, and these probably aren't on it.

Watching the BpfJailer talk helped here, because Meta ran into the same limitation. It comes from how the hooks are designed. The `inode_*` hooks don't get the mount, so there's no `struct path` to resolve.

## Walking the dentry tree

Instead of joining names to `/proc/<pid>/cwd`, I'm walking the dentry tree in the kernel, the approach SysArmor described:

1. Start from the dentry the hook gives me and follow parent pointers up toward the root, reading each name.
2. Stop at a fixed depth, since the BPF verifier only accepts bounded loops.
3. Send the names with their offsets, and resolve the mount in user space from the process's mountinfo, so the agent can place that chain in the real file system tree, including inside containers.

The tradeoff is size. A chain of names makes every rename and delete event bigger, and part 6 taught me how easy it is to overflow the ring buffer. It also no longer fits on the stack, so this is where the per-CPU scratch buffer I passed on in part 6 comes back, with one slot per hook so two hooks on the same CPU can't overwrite each other. The same work should also give long-path events a real path instead of a basename and a status flag.

Like everything since October, it'll be measured against the same baseline, with every dropped event counted. I'll write it up when it's done.
