---
title: "Learning eBPF by building a ransomware detector, part 3: the path problem"
date: 2026-11-25T10:03:00-05:00
weight: 3
series: "ebpf-build-log"
tags: ["ebpf", "linux", "lsm", "vfs"]
description: "Why the inode_rename and inode_unlink LSM hooks can't give you a full path, and what I did about it in v1."
---

Moving to LSM hooks gave me a problem I didn't expect: I couldn't get full file paths for renames and deletes.

## What I saw

At `file_open`, `bpf_d_path` gave me a clean absolute path. At `inode_rename` and `inode_unlink`, I only had the file's name, for example `report.docx`, with no directory. For a ransomware detector that's a real gap, because renames and deletes are half of what ransomware does, and a SOC analyst needs to know which `report.docx` it was.

## Digging into the root cause

To understand why, I had to learn how the kernel represents a path.

- A **dentry** is a directory entry: one name, plus a pointer to its parent dentry. Following parents gives you the chain of names up to the root of that file system.
- A **vfsmount** says where that file system is mounted in the overall tree.
- A **`struct path`** is the pair of the two. You need both to build an absolute path, because the same dentry chain can appear at different places in the tree, for example inside a container or after a bind mount.

`bpf_d_path` takes a `struct path`. The `file_open` hook gets a `struct file`, which carries one, so it works. The `inode_*` hooks are called deep in the VFS with dentries and inodes only. The mount is not passed in, so there is nothing to hand `bpf_d_path`, and the kernel only allows that helper on hooks where it knows the path is safe to read anyway. The name I could read was just the last dentry.

## The v1 workaround and why it isn't enough

For v1 I sent the name from the kernel and had the Go agent join it to the process's working directory from `/proc/<pid>/cwd`. That gives the right answer when the process works on files in its current directory, which is common.

It breaks when it matters most. A process that uses absolute paths, works in other directories, or changes directory quickly gets wrong paths, and by the time user space reads `/proc`, a short-lived process may be gone.

## The options I wrote down

When I paused in August I left this as an open question:

- **Use the `path_*` LSM hooks instead.** `path_rename` and `path_unlink` carry a `struct path`, so the mount is available. They sit higher up, though, and I'd chosen the inode hooks because they're closest to the actual change.
- **Walk the dentry chain in the kernel** and send each name to user space. It works, but every event gets much bigger, and ring buffer space was already a concern.

I came back to this later, after seeing that BpfJailer had hit the same problem ([part 6](/blog/ebpf-build-log-6/)).
