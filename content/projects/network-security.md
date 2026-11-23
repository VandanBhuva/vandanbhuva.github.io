---
title: "Network Security Infrastructure"
date: 2026-04-01
weight: 7
description: "An enterprise-style PKI with RSA-4096, a token-bucket DDoS rate limiter dropping 90% of malicious traffic, and a raw-socket Go packet sniffer."
stack: ["C", "Go", "OpenVPN", "PKI"]
---

- **Enterprise PKI (RSA-4096)** enforcing encrypted OpenVPN and OpenSSH access, with border firewalls and lateral-movement restrictions across isolated subnets.
- **Token-bucket rate limiter** that profiles per-entity traffic to mitigate DDoS, dropping **90% of malicious traffic** and capping floods at **under 2 packets/sec** without disrupting legitimate handshakes.
- **Go packet sniffer** on `AF_PACKET` raw sockets that bypasses the OS network stack, used to execute ARP poisoning and to benchmark AES-256-GCM VPN throughput at **208 Mbps**.
