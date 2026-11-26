---
title: "Learning eBPF and getting my first file events out of the kernel"
date: 2026-07-20T18:00:00-04:00
weight: 1
series: "ebpf-build-log"
tags: ["ebpf", "linux", "edr"]
description: "How I learned eBPF from the eunomia tutorials and got file events out of the kernel into a Go agent."
---

I've been learning eBPF for the past couple of weeks, and I've started a project to learn it properly: a Linux agent that spots ransomware from what it does to files and stops it while it's running. I'm going to write up the project as I go, including the parts where I get things wrong. This first post is about learning the basics and getting file events from the kernel into my own code.

I'd used eBPF a little before for a small [network threat detector](/projects/network-threat-detector/), but only at the edges. I wanted a bigger security project that went into the file system. Ransomware fits well. Whatever family it is, it has to open a lot of files, overwrite or replace them with encrypted data, and usually rename or delete the originals. That pattern of file activity is exactly what eBPF lets you watch from inside the kernel.

## What I learned from the tutorials

I worked through the [eunomia eBPF tutorials](https://eunomia.dev/tutorials/), which build up from a hello-world program to real tracing tools. The model I came away with:

- An eBPF program is a small function written in restricted C, compiled by clang into BPF bytecode, and loaded into the kernel with the `bpf()` system call.
- Before it runs, the kernel's verifier checks it. Every memory access has to be provably in bounds, loops have to be bounded, and the stack is only 512 bytes. If the verifier can't prove the program is safe, it refuses to load it. A lot of eBPF work is learning to write code the verifier can follow.
- Once verified, the program is JIT-compiled and attached to a hook: a tracepoint, a kprobe on a kernel function, a network hook, or a security hook. It runs every time that hook fires, in the context of whatever process triggered it.
- Programs can't do much on their own. They call a fixed set of helper functions (read the current PID, read kernel or user memory, get a timestamp) and keep state in maps: hash tables, arrays and per-CPU arrays that user space can read and write too.
- To get a stream of events out, you use a ring buffer or a perf buffer. Both carry records from the kernel to a user-space reader.
- BTF is type information the kernel exports about its own structs. With it you can generate a `vmlinux.h` header that has every kernel type, so a BPF program can read fields of kernel structures without kernel headers installed.

## The plumbing

Before any detection logic I needed the toolchain working end to end: `bpftool` dumping the running kernel's BTF into `vmlinux.h`, clang building the BPF object, and a loader in user space.

For hooks I started with syscall tracepoints, since they were the easiest way to see file activity while I was still learning. The first program attached to `sys_enter_openat` and reported every open, with the process ID, its command name, and the filename. The filename is a pointer into the calling process's memory, so the program copies it with `bpf_probe_read_user_str` into a fixed-size buffer in the event. I added `sys_enter_renameat2` next, since ransomware renames files as much as it writes them.

To move events to user space I used a BPF ring buffer rather than a perf buffer. A perf buffer is one buffer per CPU, so events from different CPUs arrive separately and can come out of order. The ring buffer is one buffer shared by all CPUs, keeps events in order, and lets the program reserve space for a record, fill it in place and submit it, with no extra copy.

For the user-space side I chose Go, using `bpf2go` from the cilium/ebpf library. It compiles the C program and generates Go code with typed handles for every program and map in it, so the agent can load the object, attach the hooks and read the ring buffer without hand-written glue. Each record is a fixed-layout C struct, and the Go side decodes it into a matching Go struct.

## First bug

The first real bug was in string handling. Filenames come across as fixed-size C char arrays, and Go printed the whole array, including whatever bytes sat after the null terminator, so names had junk on the end. The fix was to cut each string at the first zero byte when decoding. It was a small bug, but it taught me to think of an event as raw bytes with a layout, not as the strings I wanted it to be.

## Where this leaves me

I now have a pipeline from a kernel hook to a Go program. What I don't have is confidence that tracepoints are the right hooks for a security tool. That's the next post.

## Resources

- [eunomia eBPF tutorials](https://eunomia.dev/tutorials/): the tutorials I learned from, from hello world up to ring buffers and LSM programs.
