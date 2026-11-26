---
title: "Learning eBPF by building a ransomware detector, part 8: what was actually written?"
date: 2026-11-11T18:00:00-05:00
weight: 8
series: "ebpf-build-log"
tags: ["ebpf", "lsm", "entropy", "edr"]
description: "Windowed indicators, checking file content after the write instead of before, auditing exclusions, and catching renames over existing files."
---

*November 9 to 11, 2026.*

The second assumption v1 made was that it knew what a process had written. Most of my false positives and most of the ways around the detector came from here.

## One rename shouldn't be a kill

The ransom-note and extension signals were flags that never expired, worth 80 and 100 points. One `mv backup.tar backup.tar.enc` scored 103 and was killed. `readme` was on the ransom-note list as a substring, so `README.md`, `README.Debian` and dpkg's `README.md.dpkg-new` all counted, and about 20 writes later velocity pushed `tar` or `dpkg` over 100. Killing dpkg in the middle of unpacking can corrupt the package database.

Both signals are now counted over a 30-second window, like the others. An extension needs three renames in the window before it scores. A ransom note has to be a newly created file, with the same note name in three or more directories, alongside renames or encryption in the same window. Real ransomware leaves a note in each directory it encrypts.

Telling "newly created" from "rewritten" can't be done reliably from user space after the fact, but the kernel knows: it sets `FMODE_CREATED` on the file before calling the `file_open` hook. That flag isn't in `vmlinux.h`, so I define it myself. I added a flags field to the event to carry it up, laid out so nothing else moved and no padding appeared. The cost is that my note-dropping test sim is now contained at its third rename instead of its first, two files later.

## Checking content after the write

This was the question I'd left open in August. v1 checked a file's entropy when it was opened for writing, which is before the new bytes exist. An in-place overwrite was read while it still held plaintext, and one of my demos was only caught because Python's `"wb"` truncates the file first. It took one 4 KB sample from the start of the file against a fixed threshold, so encrypting only parts of a file stayed under it, and memory-mapped writes produced no event at all. The worker also opened the written path with a plain `os.Open`, which a FIFO can block forever.

I needed to know when a file was written and when the writing was done.

- Writes: `lsm/file_permission`, filtered to writes. It covers `write`, `pwrite`, `writev`, `splice` and io_uring, because they all go through the same permission check. I confirmed io_uring live with a test that makes zero write syscalls under strace. `bpf_d_path` isn't allowed in this hook, but the `bpf_path_d_path` kfunc (Linux 6.12) is allowed in every LSM hook, so I raised the minimum kernel to 6.12.
- Memory-mapped writes: `lsm/mmap_file`, for shared writable mappings.
- Done: `lsm/file_release`, when the file is closed for the last time. This was the fix I'd guessed at in August.

Sending one event per write would be about 262,000 events per gigabyte written in 4 KB chunks. An LRU map in the kernel keyed by process and file sends only the first write and updates a timestamp for the rest. Writing 1 GB in 4 KB chunks deduped 262,265 writes while the agent saw single-digit events per second.

When a file is done, user space compares it before and after. The "before" is the hard part: by the time the agent reads it, a fast encryptor has often already written, and reading ciphertext as "before" would make the change invisible. So I only trust a "before" read when the file wasn't created or truncated by the open and its ctime (which, unlike mtime, can't be set back) is from before the open. Otherwise the file's extension says what format it should be.

Entropy alone can't separate compression from encryption. Measured on 4 KB samples, the middle of an xz or gzip file looks as random as ciphertext, and only file starts differ. The header is what gives compressed data away. So the verdict goes in order: did the format's header disappear or change, did text stop being text, did entropy rise at the same offset, and is the content near uniform (entropy plus a chi-square test) with no recognizable header. `gzip`, `xz` and `zstd` keep valid headers and don't get flagged.

Files are now opened safely: first by `O_PATH`, then `fstat` to check it's a regular file with the inode the kernel reported, then reopened non-blocking. That closes the FIFO stall, and with the inode check the agent can read through the writer's own root (`/proc/<pid>/root`), which is a container's view of the file system.

The benchmark found a race I hadn't thought of. Outbreak runs were contained 8 of 10 times. In the two misses, the simulation had renamed each file before the worker reached its open, so opening by path failed and there was no content verdict. I fixed it by following the process's renames by inode. Each written file was also now 5 events instead of 3, so I raised the ring buffer from 256 KB to 1 MB.

In-place, partial-chunk and mmap encryption are now all contained after 2 files. gzip of 8 files scored 104 before and 24 now.

## Every exclusion is a place to hide

I audited every exclusion. The database fix from v1 hadn't held: every event for `.db`, `.sqlite` and `-wal` files was still dropped, and a rename only checked the old name, so encrypting `customers.db` into `customers.db.lockbit` scored nothing. Path exclusions like `/snap/` matched anywhere, so any folder of mine called `snap` wasn't monitored. And because the attacker chooses the output file name, a suffix exclusion like `.o` hid any ciphertext saved under that name.

The path and database skips are gone. Browsers and editors are already skipped by their verified executable, and the content check makes the rest harmless. The few build-artifact suffixes I kept only stop velocity counting, so a compile writing hundreds of `.o` files doesn't look fast. Those files still get a content check, since ciphertext saved as `.o` should still be judged. I then checked SQLite live: WAL and journal churn on five databases with random blobs stays at velocity 30, and twenty databases overwritten in place with ciphertext are contained.

Two days earlier the agent had killed `gnome-shell` during a test run, which logged me out. It writes a temp file and deletes it, and with the wrong directory guessed for the delete, that looked like write-then-delete. The desktop session binaries are now protected, under the same root-owned, same-file rules as the allowlist.

## Renaming over a file is a delete

Ransomware doesn't have to delete anything. Writing ciphertext to `contract.pdf.tmp` and renaming it over `contract.pdf` destroys the original, and the kernel never calls `inode_unlink`. That pattern scored 90 and was never contained. Hard links had a similar gap.

`inode_rename` runs with the target already looked up, so if the target dentry has an inode, a regular file is about to be replaced. The event now carries that as a flag, plus the inode of the file being renamed. A new `inode_link` hook covers links. In user space, a rename moves the write record and its "encrypted" mark to the new name by inode, so whether the content verdict lands before or after the rename, the replace scores once, as the same loss as an encrypted copy plus a delete. An editor's atomic save (text renamed over the original) still scores 30.

With this, the last of the known-bad tests from October passed.
