---
title: "Computation-Efficient Private Access Control Lists"
date: 2024-12-01
weight: 3
description: "Research: a private access control scheme for Function Secret Sharing that swaps modular exponentiation for LtHash homomorphic hashing, cutting execution time by ~99%. Poster at IEEE CCNC 2026."
stack: ["Applied cryptography", "Function Secret Sharing", "LtHash"]
---

> Research internship at DA-IICT with **Dr. Maniklal Das**, May–December 2024. Accepted as a **poster at IEEE CCNC 2026**.

## Background

**Function Secret Sharing (FSS)** splits a function into shares held by several evaluators. Each share alone reveals nothing, but together they compute the function. It underpins private information retrieval, private statistics, and anonymous messaging.

Those systems need **access control**: evaluators must confirm that a dealer is allowed to touch a function *without learning which function it is*. Servan-Schreiber et al. (IEEE S&P 2023) proposed a private access control list (PACL) for this, but its audit step relies on modular exponentiation, which is expensive.

## Contribution

We kept the building blocks of the original scheme and replaced the expensive part:

- A **Key Distribution Centre** generates each function's access key, splits it into shares, and publishes a verification key computed as the **LtHash** of those shares.
- Because LtHash is **set-homomorphic**, each evaluator can compute an audit token with plain multiplications and additions. The tokens sum to zero exactly when the dealer holds the right key.
- That turns **N modular exponentiations into N multiplications**, reducing audit cost from **O(N log b)** to **O(N)**, and cut execution time by **~99%** compared with the previous implementation.

The paper also analyzes the security and efficiency of the scheme.
