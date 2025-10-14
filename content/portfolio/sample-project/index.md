---
title: "Distributed Consensus Protocol"
date: 2025-01-15
draft: false
description: "A fault-tolerant consensus protocol implementation for distributed systems"
technologies: ["Rust", "Networking", "Cryptography", "Consensus"]
github_url: "https://github.com/phreaknik/consensus-protocol"
demo_url: ""
featured_image: "architecture.png"
weight: 1
showToc: true
---

## Overview

This project implements a Byzantine fault-tolerant consensus protocol designed for high-throughput distributed systems. The protocol ensures safety and liveness properties even when up to 1/3 of nodes are malicious or offline.

## Technologies Used

- **Rust**: Core implementation language for performance and safety
- **Networking**: Custom UDP-based networking layer for low-latency communication
- **Cryptography**: Ed25519 signatures and Blake3 hashing for security
- **Consensus**: Novel BFT algorithm with optimistic fast path

## Key Features

- Sub-millisecond consensus latency in optimal conditions
- Handles network partitions and Byzantine failures
- Configurable safety vs. liveness trade-offs
- Built-in benchmarking and simulation tools

## Implementation Details

The consensus protocol uses a leader-based approach with view changes for fault tolerance. The key innovation is an optimistic fast path that bypasses the traditional prepare phase when network conditions are favorable.

### Architecture

The system consists of several key components:

- **Node Manager**: Handles peer discovery and connection management
- **Consensus Engine**: Core BFT algorithm implementation
- **Network Layer**: High-performance message passing
- **State Machine**: Pluggable application state transitions

### Performance Optimizations

- Zero-copy message serialization using custom binary format
- Batching of consensus instances for improved throughput
- Adaptive timeout mechanisms based on network conditions
- Memory pool allocation for reduced garbage collection

## Results

Achieved 100,000+ transactions per second with 4 nodes on a local network, with consensus latency under 1ms in optimal conditions. The implementation successfully handles various fault scenarios including:

- Random node crashes and recoveries
- Network partitions and healing
- Byzantine behavior simulation
- Performance degradation under load

## Links

- [Source Code](https://github.com/phreaknik/consensus-protocol)
- [Performance Benchmarks](https://github.com/phreaknik/consensus-protocol/tree/main/benchmarks)