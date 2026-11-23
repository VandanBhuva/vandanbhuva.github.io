---
title: "Systems Engineering: Allocators, Caches & Distributed Consensus"
date: 2026-04-15
weight: 7
description: "The systems work underneath the security work: a 64-bit malloc, a cache simulator, Raft-replicated key-value and database clusters, and a reflection-based RPC framework."
stack: ["C", "Go", "Raft"]
---

Security engineering depends on knowing how systems actually behave. These projects are where I built that foundation.

## Dynamic memory allocator (C)

A 64-bit `malloc` / `free` / `realloc` / `calloc` returning 16-byte aligned pointers with **under 128 bytes of global overhead**. Segregated free lists and boundary-tag coalescing without footers on allocated blocks, tuned to reach **over 74% peak memory utilization** while keeping throughput high.

## Cache simulator (C)

A parameterized cache simulator with LRU replacement and write-back / write-allocate. Used it to optimize a matrix transpose with cache blocking, getting under 36,000 cycles (direct-mapped) and 35,100,000 cycles (8-way L1) for 32×32 and 1024×1024 matrices.

## Distributed database platform (Go)

Consistent hashing for horizontal and vertical partitioning across independent **3-node Raft clusters**, with a coordinator metadata server and a 200-entry LRU read cache routing SQL and NoSQL CRUD operations. Load-tested for **zero data loss** and strict consistency through leader failures under 8 concurrent clients.

## Hierarchical key-value cluster (Go)

Directory paths sharded across up to **5 Raft consensus groups**, with randomized 400–800 ms election timeouts and 150 ms heartbeats keeping commits consistent through network partitions. Exposed through an idempotent 6-endpoint HTTP API.

## RPC framework (Go)

A thread-safe, generic RPC library that uses reflection to bind client stubs to remote methods without type registration, with a custom byte-blob encoder for nested data and at-least-once delivery over lossy networks.
