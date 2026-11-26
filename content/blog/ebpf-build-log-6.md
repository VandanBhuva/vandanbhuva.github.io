---
title: "eBPF ransomware monitor, part 6: checking what was actually written"
date: 2026-11-25T10:06:00-05:00
weight: 6
series: "ebpf-build-log"
tags: ["ebpf", "linux", "lsm", "edr"]
description: "Rewriting the content check around write and close hooks took false positives from 6 of 12 runs to 0 of 24."
---

*November 9 to 11, 2026.*

The entropy check read a file when it was opened for writing, before the new bytes existed. An in-place `O_RDWR` overwrite was read as its old plaintext, `mmap` writes produced no event at all, and a FIFO opened by a worker could block it forever.

## The rewrite

The rewrite added three kernel hooks: `file_permission` for writes (covering `write`, `pwrite`, `writev`, `splice` and io_uring), `mmap_file` for shared writable mappings, and `file_release` for the final close. The agent now reads a file's "before" state at open and its "after" state at close.

It checks the format first (a PDF or zip that lost its header, or text that stopped being text), then looks for an entropy rise at the same offset, and finally tests for near-uniform content with entropy plus a chi-square test. Compressed output keeps a valid header, so `gzip`, `xz` and `zstd` pass. `test/write_path.py` covers 14 live cases, including a raw-syscall io_uring encryptor.

## Two more fixes

- **Exclusion audit.** Blanket skips for databases and paths like `/snap/` were attack surface, since the attacker picks the file name. They were removed, and the desktop session was added to the protected list after the November 9 kill of `gnome-shell`.
- **Rename over a file.** Renaming ciphertext onto an existing file destroys it without an unlink. Rename and link events now carry the inode, so that case and "link then delete" both correlate.

## Results

Re-running the baseline on the same day against both builds showed no regression in files preserved (outbreak median 985 vs 983.5 of 1,000) and false positives down from 6 of 12 runs to 0 of 24. Every lost event is now counted, where the baseline lost up to 64% silently. I tagged this `phase1-done`.

## What's next

The code hasn't changed since November 11. The next round moves from killing a process after detection to blocking its writes in the kernel, followed by a final benchmark comparing every version. I'll write that up when it's done.
