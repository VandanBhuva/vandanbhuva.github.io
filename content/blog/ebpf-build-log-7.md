---
title: "Learning eBPF by building a ransomware detector, part 7: which process is this?"
date: 2026-11-25T10:07:00-05:00
weight: 7
series: "ebpf-build-log"
tags: ["ebpf", "task-storage", "linux", "edr"]
description: "Moving process identity into the kernel with task storage, seeding it with a task iterator, and making containment safe against PID reuse."
---

*October 30 to November 9, 2026.*

v1 identified processes by name and by PID, and both are weaker than they look. These three items are where BpfJailer's design helped most.

## Names can be faked

My allowlist skipped processes named `node`, `git` or `code`, and containment refused to touch anything named `sshd` or `systemd`. That name is `comm`, which a process can set to anything with one `prctl(PR_SET_NAME)` call. On the old agent, a test encryptor that renamed itself got its opens skipped entirely, and one calling itself `sshd` reached CRITICAL and was refused 51 times. 0 of 50 files survived in each case.

Identity now lives in the kernel. BPF task storage holds each task's executable as a device and inode pair. It's set at exec and copied to children at fork, the way BpfJailer tracks processes. I set it in `bprm_committed_creds`, which runs once, after the point where an exec can no longer fail. `bprm_check_security` runs before that, and once per binary format pass (a script, then its interpreter), so a failed exec could have changed the identity.

I kept the identity as (device, inode) and not a path or hash in the kernel, because paths aren't unique (bind mounts, namespaces) and hashing at hook speed isn't possible. User space resolves the key once per process through `/proc/<pid>/exe` and checks with `fstat` that it's the same file the kernel saw. The allowlist trusts a path only for a root-owned, non-writable file that the same path reaches from the agent's own mount namespace. Anything else, like tools in `~/go/bin`, has to be allowlisted by SHA-256. Hashing runs in the background, because a 210 MB binary would stall a worker for about 0.3 seconds, and until a hash is in, the process is scored as untrusted.

All three bypasses went from 0 to 49 of 50 files saved. Kernel threads used to be protected by name too; they're now recognized by the `PF_KTHREAD` flag.

## Processes that started first

Task storage only fills when something execs or forks. Anything already running when the agent started, including malware that got in first, had no kernel identity and fell back to `/proc`, which is read late and can be wrong if the process has exec'd since.

This was the SysArmor idea: a BPF task iterator that walks every thread once at startup and fills in task storage from each task's mapped executable. The order matters. I run it after the exec and fork hooks attach and before the file hooks do, so a task that execs during the walk is caught by the hooks, and no file event is reported before the walk is done. The walk never overwrites an entry, since an existing one was set by an exec that happened after the hooks attached and is newer than what the walk read.

On my VM it seeds about 850 tasks. A test encryptor started 8 seconds before the agent now gets its identity from the kernel, and across a full benchmark no process fell back to `/proc`. There is still a way for a process that starts before the agent to misreport its executable in some setups. I've confirmed it's possible on my kernel, and closing it is still on the list.

## PIDs get reused

Containment was keyed by PID, and Linux hands a PID out again once its process exits. That caused three bugs. A process that later got a contained process's PID was never contained. A process that exited left its score to whoever got its PID next. And if a process exited between detection and the signal, the signal could hit the PID's new owner.

To test this without root, I wrote a small C program that forks until the PID counter wraps around to a chosen PID. Parallel forkers get close, then single forks land on it, about 4 minutes per wrap with the agent running. On the old agent, an innocent process that inherited a PID started at score 81 and was killed on its 20th plain-text write. A malicious one that inherited a contained PID was never contained.

Now the kernel stamps each event with the process's start time (`start_boottime` of the thread group leader, the same value `/proc/<pid>/stat` reports), and every piece of per-process state is keyed by PID and start time. Containment opens a pidfd first, then checks the start time in `/proc`. If it matches, the pidfd refers to the scored process and keeps referring to it even after it exits, so `SIGSTOP` and `SIGKILL` sent through it can't reach anyone else. If the process changed its executable since it was scored, it's killed anyway and the snapshot records the new binary. Refusing would let malware escape by exec'ing `sshd`.

The innocent reuser now peaks at 25 and exits normally, and the malicious one is contained with 49 of 50 files saved.

During one of these runs the agent killed the host's own logrotate, at score 110: rotating and compressing logs looks like writing a file and deleting the original, plus high entropy from gzip. That was the next problem.
