---
title: "Offensive Security & Exploit Development"
date: 2026-10-01
weight: 4
description: "A working path-traversal exploit for CVE-2026-113613 in FlexGet, cryptographic and memory-corruption exploits, and multi-segment network penetration tests."
stack: ["Python", "C", "Penetration testing"]
---

## Weaponizing a disclosed CVE

CVE-2026-113613 in FlexGet had been disclosed but had no public exploit. I built a working **path-traversal exploit** that achieves **arbitrary file write outside the sandboxed output directory**.

{{< todo >}}This is a great candidate for your first writeup. A post on how you went from advisory to working exploit would show your process well.{{< /todo >}}

## Cryptographic attacks

- AES **padding oracle** attack.
- SHA-256 **length extension** attack.

## Memory corruption

Buffer overflows and **ROP chains** to bypass NX and ASLR.

## Penetration testing

End-to-end penetration tests on multi-segmented environments, reaching the highest privileges across **12+ enterprise-grade hosts**.
