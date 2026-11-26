---
title: "The path problem"
date: 2026-09-08T19:00:00-04:00
weight: 4
series: "ebpf-build-log"
tags: ["ebpf", "linux", "vfs", "lsm"]
description: "Why the inode rename and unlink LSM hooks only give you a file name, the kernel history behind it, and the tradeoffs between the ways around it."
---

Moving to LSM hooks gave me a problem I didn't expect, and it's taken more of my time than anything else in this project: I can't get full file paths for renames and deletes.

## What I saw

At `file_open`, `bpf_d_path` gives me a clean absolute path. At `inode_rename` and `inode_unlink` I only have the file's name, for example `report.docx`, with no directory. For a ransomware detector that's a real gap. Renames and deletes are half of what ransomware does, and whoever reads the alert needs to know which `report.docx` it was.

## The root cause

To understand why, I had to learn how the kernel represents a path.

- A **dentry** is a directory entry: one name, plus a pointer to its parent dentry. Following parents gives you the chain of names up to the root of that file system.
- A **vfsmount** says where that file system is mounted in the overall tree.
- A **`struct path`** is the pair of the two. You need both for an absolute path, because the same dentry chain can appear at different places in the tree, for example inside a container or after a bind mount.

`bpf_d_path` takes a `struct path`. The `file_open` hook gets a `struct file`, which carries one, so it works there. The `inode_*` hooks are called from `vfs_rename` and `vfs_unlink`, deep in the VFS, with dentries and inodes only. The mount isn't passed down that far, so there's nothing to hand `bpf_d_path`. All I can read is the last dentry's name, with `bpf_probe_read_kernel_str`.

## Why the inode hooks don't get the mount

The obvious fix is to pass the vfsmount down into `vfs_unlink` and `vfs_rename`, so the inode hooks have it. That turns out to be an old argument. Around 2008, path-based security modules like AppArmor and TOMOYO wanted exactly this, and the VFS developers didn't want mount information threaded through the inode-level functions ([LWN covered it](https://lwn.net/Articles/277833/)). The compromise was a second set of hooks, the `security_path_*` hooks, which are called higher up where the `struct path` is still available.

So the kernel has two sets of hooks for the same operations:

- `path_rename` and `path_unlink` carry a `struct path`, so a full path is possible (with the `bpf_path_d_path` kfunc). But not every caller goes through them. Some kernel-internal callers and some overlayfs operations call the VFS functions directly and only reach the inode hooks ([a 2019 LSM mailing list thread](https://kernsec.org/pipermail/linux-security-module-archive/2019-April/012583.html) discusses this). For a security tool, that's a coverage gap.
- `inode_rename` and `inode_unlink` see every rename and unlink at the lowest VFS layer, but without the mount.

## Asking around

I reached out to a couple of people who work on eBPF security. One suggestion was simple: 2008 was a long time ago, so write a patch, send it to the VFS and BPF mailing lists, and see what the maintainers think now. That's something I'd like to do.

The more useful answer changed how I think about the problem. Even if the inode hooks had the vfsmount, a path built from it can't be fully trusted. Any unprivileged user can create their own user and mount namespace, bind mount a directory somewhere new, and make a file appear under whatever path they like. A path that the attacker can choose is weak evidence for a detector, and useless for an access control system. It's also the reasoning behind Meta's BpfJailer working from the inode hooks and resolving paths itself.

I then found the [BpfJailer talk from Linux Plumbers 2025](https://lpc.events/event/19/contributions/2159/) and went through the slides. They walk the dentry tree inside the kernel, and the reason isn't only performance. Sending raw dentries to user space and rebuilding the path there opens a race: by the time user space reads the tree, it may have changed. Walking it inside the hook gives a consistent answer for that moment. It was honestly reassuring to see experienced engineers hit the same wall. I only wanted absolute paths for my SIEM logs and didn't expect it to go this deep.

## The workaround I have for now

For now the kernel sends the name, and the Go agent joins it to the process's working directory from `/proc/<pid>/cwd`, with `/proc/<pid>/fd` as another source. That gives the right answer when a process works on files in its current directory, which is common.

It breaks when it matters most. A process that uses absolute paths, works in other directories, or changes directory quickly gets wrong paths, and by the time user space reads `/proc` a short-lived process may already be gone.

## The options

- Switch to the `path_*` hooks and accept the coverage gap.
- Walk the dentry chain in the kernel, inside the inode hook. This keeps full coverage and avoids the race, but it means fighting the verifier on a pointer-chasing loop, which has to be bounded, and a much bigger event for every rename and delete.
- Send each dentry to user space and rebuild the path there, as in [this Stack Overflow answer](https://stackoverflow.com/questions/79631612/how-can-i-safely-build-the-full-path-of-a-struct-dentry-in-an-ebpf-lsm-hook). It's easier to write, but adds overhead per event and has the race above.
- Keep a shadow of the file system in user space: listen to directory creation and mount events and keep an in-memory tree in Go, so any dentry can be looked up without asking the OS. It's a lot of state to keep correct.

I'm leaning toward the dentry walk in the kernel, keeping the inode hooks for coverage. I'm pausing the project for some time, with the cwd workaround in place, and this as the first thing to fix when I come back.

## Resources

- [LWN: the 2008 discussion of passing vfsmount to the LSM inode hooks](https://lwn.net/Articles/277833/)
- [linux-security-module archive: callers that skip the path hooks](https://kernsec.org/pipermail/linux-security-module-archive/2019-April/012583.html)
- [Stack Overflow: building the full path of a struct dentry in an eBPF LSM hook](https://stackoverflow.com/questions/79631612/how-can-i-safely-build-the-full-path-of-a-struct-dentry-in-an-ebpf-lsm-hook)
- BpfJailer: eBPF based Mandatory Access Control, Linux Plumbers 2025: [talk page](https://lpc.events/event/19/contributions/2159/), [video](https://www.youtube.com/watch?v=o78vnXfih8E), [slides](https://lpc.events/event/19/contributions/2159/attachments/1833/3929/BpfJailer%20LPC%202025.pdf)
