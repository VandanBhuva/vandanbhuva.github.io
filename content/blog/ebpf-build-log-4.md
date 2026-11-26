---
title: "Learning eBPF by building a ransomware detector, part 4: the path problem"
date: 2026-09-08T19:00:00-04:00
weight: 4
series: "ebpf-build-log"
tags: ["ebpf", "linux", "vfs"]
description: "Why the inode rename and unlink LSM hooks only gave me a file name, what a path is inside the kernel, and the workaround I shipped."
---

*July to August, 2026.*

Moving to LSM hooks gave me a problem I didn't expect: I couldn't get full file paths for renames and deletes.

## What I saw

At `file_open`, `bpf_d_path` gave me a clean absolute path. At `inode_rename` and `inode_unlink` I only had the file's name, for example `report.docx`, with no directory. For a ransomware detector that's a real gap. Renames and deletes are half of what ransomware does, and an analyst needs to know which `report.docx` it was.

## The root cause

To understand why, I had to learn how the kernel represents a path.

- A **dentry** is a directory entry: one name, plus a pointer to its parent dentry. Following parents gives you the chain of names up to the root of that file system.
- A **vfsmount** says where that file system is mounted in the overall tree.
- A **`struct path`** is the pair of the two. You need both for an absolute path, because the same dentry chain can appear at different places in the tree, for example inside a container or after a bind mount.

`bpf_d_path` takes a `struct path`. The `file_open` hook gets a `struct file`, which carries one, so it works there. The `inode_*` hooks are called deep in the VFS with dentries and inodes only. The mount isn't passed in, so there is nothing to hand `bpf_d_path`. On top of that, the kernel only allows `bpf_d_path` from a fixed allowlist of hooks where it knows the path is safe to read. The name I could read was just the last dentry, with `bpf_probe_read_kernel_str`.

## The workaround I shipped

For v1 the kernel sent the name, and the Go agent joined it to the process's working directory read from `/proc/<pid>/cwd`, with `/proc/<pid>/fd` as another source. That gives the right answer when a process works on files in its current directory, which is common.

It breaks when it matters most. A process that uses absolute paths, works in other directories, or changes directory quickly gets wrong paths, and by the time user space reads `/proc` a short-lived process may already be gone. My own test scripts all used absolute `/tmp` paths outside their working directory, so their rename and delete paths were wrong. Correlation still worked only because it compared file stems, not full paths.

## The options I wrote down

When I paused in August I left this as an open question:

- Use the `path_*` LSM hooks instead. `path_rename` and `path_unlink` carry a `struct path` for the directory, so the mount is available. I'd chosen the inode hooks because they sit closest to the actual change, and I wasn't sure when to use which.
- Walk the dentry chain in the kernel and send each name to user space. I found a [Stack Overflow answer](https://stackoverflow.com/questions/79631612/how-can-i-safely-build-the-full-path-of-a-struct-dentry-in-an-ebpf-lsm-hook) that does roughly this, but it sends each dentry's information up to user space, which adds a lot of overhead per event.
- Keep a shadow of the file system in user space. Listen to directory creation and mount events and keep an in-memory tree in Go, so any dentry can be looked up without asking the OS.

I came back to this in November, after the BpfJailer talk showed me Meta had hit the same problem ([part 10](/blog/ebpf-build-log-10/)). Before that, the bad paths caused a bug I'd blamed on scoring ([part 5](/blog/ebpf-build-log-5/)).
