# Distributed Systems Concepts - In-Depth Learning Guide

This folder contains comprehensive explanations of distributed systems concepts, organized by category. Each concept is explained with:

- **What it is** - Clear definition and overview
- **The Analogy** - Real-world comparison for intuition
- **Why it exists** - The problem it solves
- **How it works** - Detailed mechanism with diagrams
- **When to use it** - Good fit / Avoid when
- **How to use it effectively** - Best practices
- **Interview Tips** - Key points to mention

---

## Folder Structure

```
concepts/
├── 01-architectural-patterns/       # System design patterns (14 files)
│   ├── backpressure.md
│   ├── bulkhead-pattern.md
│   ├── circuit-breaker.md
│   ├── cqrs-event-sourcing.md
│   ├── dual-write-problem.md
│   ├── event-carried-state-transfer.md
│   ├── leader-follower-replication.md
│   ├── lmax-disruptor.md
│   ├── orchestration-vs-choreography.md
│   ├── saga-pattern.md
│   ├── seda.md
│   ├── sidecar-pattern.md
│   ├── strangler-fig-pattern.md
│   ├── temporal-workflow.md
│   └── transactional-outbox.md
│
├── 02-distributed-algorithms/       # Core algorithms (18 files)
│   ├── bloom-filters.md
│   ├── byzantine-fault-tolerance.md
│   ├── consensus-algorithms.md
│   ├── consistent-hashing.md
│   ├── distributed-locks.md
│   ├── gossip-protocol.md
│   ├── hedged-requests.md
│   ├── leader-election.md
│   ├── load-shedding.md
│   ├── mapreduce.md
│   ├── quorum.md
│   ├── rate-limiting-algorithms.md
│   ├── reservoir-sampling.md
│   ├── sharding.md
│   ├── split-brain.md
│   ├── tail-latency.md
│   ├── two-phase-commit.md
│   └── vector-clocks.md
│
├── 03-data-structures/              # Data structures (6 files)
│   ├── b-trees.md
│   ├── crdts.md
│   ├── hyperloglog.md
│   ├── lsm-trees.md
│   ├── merkle-trees.md
│   └── skip-lists.md
│
├── 04-low-level-design/             # LLD patterns (6 files)
│   ├── builder-pattern.md
│   ├── design-patterns-overview.md
│   ├── factory-pattern.md
│   ├── observer-pattern.md
│   ├── singleton-pattern.md
│   └── strategy-pattern.md
│
├── 05-system-design/                # HLD concepts (15 files)
│   ├── api-design.md
│   ├── caching-strategies.md
│   ├── cdn-edge-computing.md
│   ├── client-side-load-balancing.md
│   ├── database-scaling.md
│   ├── distributed-tracing.md
│   ├── idempotency.md
│   ├── layer4-vs-layer7.md
│   ├── load-balancing.md
│   ├── message-queues.md
│   ├── message-queues-vs-worker-pools.md
│   ├── microservices-architecture.md
│   ├── quic-protocol.md
│   ├── retry-strategies.md
│   └── rpc-vs-rest.md
│
├── 06-concurrency/                  # Concurrency (3 files)
│   ├── locks-synchronization.md
│   ├── race-conditions.md
│   └── thread-pools.md
│
├── 07-llm-optimization/             # LLM optimization (14 files)
│   ├── cpu-offloading.md
│   ├── flash-attention.md
│   ├── gradient-checkpointing.md
│   ├── kv-cache.md
│   ├── lora.md
│   ├── mixed-precision.md
│   ├── parameter-efficient-finetuning.md
│   ├── pruning-distillation.md
│   ├── quantization.md
│   ├── retrieval-augmented-compression.md
│   ├── sharded-training.md
│   ├── sparse-moe.md
│   ├── speculative-decoding.md
│   └── weight-sharing.md
│
└── 08-llm-memory/                   # LLM Memory Systems (12 files)
    ├── attention-mechanisms.md
    ├── differentiable-neural-computers.md
    ├── episodic-memory.md
    ├── forgetting-mechanisms.md
    ├── hierarchical-memory.md
    ├── lifelong-learning-memory.md
    ├── local-global-memory-fusion.md
    ├── long-context-attention.md
    ├── memory-augmented-transformers.md
    ├── recurrent-memory-layers.md
    ├── retrieval-augmented-memory.md
    ├── sliding-window-attention.md
    ├── vector-databases.md
    └── working-memory-buffers.md
```

---

## How to Use This Guide

1. **Start with fundamentals** - Begin with architectural patterns and distributed algorithms
2. **Practice with implementations** - Each concept includes code examples
3. **Connect concepts** - Understand how patterns work together in real systems
4. **Apply in interviews** - Use the Interview Tips section for discussions

---

## Quick Reference by Interview Type

### DSA Interviews
- `03-data-structures/` - B-Trees, Skip Lists, LSM Trees, CRDTs, HyperLogLog

### LLD Interviews
- `04-low-level-design/` - Design patterns (Factory, Strategy, Observer, etc.)

### HLD/System Design Interviews
- `01-architectural-patterns/` - CQRS, Saga, Bulkhead, Circuit Breaker
- `02-distributed-algorithms/` - Consistent Hashing, Consensus, Quorum
- `05-system-design/` - Idempotency, Caching, Message Queues, Load Balancing

### Concurrency Interviews
- `06-concurrency/` - Locks, Race Conditions, Thread Pools

### LLM/AI System Design
- `07-llm-optimization/` - LoRA, Quantization, KV-Cache, Flash Attention
- `08-llm-memory/` - RAG, Vector DBs, Attention Mechanisms

