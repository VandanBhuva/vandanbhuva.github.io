---
title: "Learning eBPF by building a ransomware detector, part 2: moving to LSM hooks"
date: 2026-07-24T18:00:00-04:00
weight: 2
series: "ebpf-build-log"
tags: ["ebpf", "linux", "lsm", "edr"]
description: "Why syscall tracepoints and kprobes weren't a safe base for a security tool, and what BPF LSM hooks give you instead."
---


## Why tracepoints weren't enough

Tracepoints were fine for watching, but the more I read about how security tools use eBPF, the less they looked like the right base. Three things bothered me.

A syscall tracepoint sees the arguments the way the program passed them, including a pointer to a path in user memory. A malicious process can change that memory after my probe reads it and before the kernel uses it. That's a time-of-check to time-of-use (TOCTOU) race, and it means the path I logged might not be the file that was actually touched.

Syscalls also aren't the only way into a file operation. `openat2` and io_uring reach the same kernel code without going through the syscall I was hooking.

And a tracepoint can only observe. If I ever wanted to block an operation instead of reacting to it, I'd need a hook that can return an error.

I'd planned to use kprobes on internal kernel functions for some of this, but decided against it. Those functions have no stable interface and can change between kernel versions, which is a bad property for something meant to run on many machines.

## LSM hooks

So I moved to BPF LSM hooks: `file_open`, `inode_rename` and `inode_unlink`. The Linux Security Module hooks are the points where the kernel asks "should this be allowed?" They run inside the VFS after the kernel has copied the arguments in and resolved them to its own objects (a `struct file`, a dentry, an inode). That gave me three things at once:

- the user-memory race is gone, because I read kernel objects, not user pointers;
- every path into the operation goes through the hook, whichever syscall started it;
- the hook can return an error, so denying an operation in the kernel is possible later.

Tetragon makes the same choice. The cost is that BPF LSM needs a 5.7 or newer kernel and `bpf` in the `lsm=` boot parameter, which I checked for at startup.

I also adopted CO-RE (`BPF_CORE_READ` against `vmlinux.h`), so one compiled binary runs across kernel versions with different struct layouts, without kernel headers on the target machine.

The `file_open` hook only passes write opens (`O_WRONLY` or `O_RDWR`) and drops pseudo-filesystems like `/dev`, `/proc` and `/run` before anything reaches user space. Crossing from the kernel to user space is the expensive part, so the less that crosses, the better.

Next I'm building the actual detection on top of these three hooks.
