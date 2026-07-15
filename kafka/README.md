# Kafka — Interview Preparation

> Core Kafka concepts for a senior/10-YOE interview: architecture and storage internals, producers, consumers and consumer groups, delivery semantics and reliability, retention/compaction/performance. Every section has a worked example and an explicit pitfall; every file ends with an Interview Q&A section.

## Contents

| # | Part | Covers |
|---|---|---|
| 1 | [Architecture & Core Concepts](./Part1-Architecture-and-Core-Concepts.md) | Brokers, topics, partitions, the commit log model, offsets, replication, leader/follower, ISR, controller election |
| 2 | [Producers](./Part2-Producers.md) | Partitioning strategy, `acks`, batching/`linger.ms`, idempotent producer, producer retries and ordering |
| 3 | [Consumers & Consumer Groups](./Part3-Consumers-and-Consumer-Groups.md) | Consumer groups, partition assignment, rebalancing (eager vs cooperative), offset commit strategies, `max.poll.interval.ms` |
| 4 | [Delivery Semantics & Reliability](./Part4-Delivery-Semantics-and-Reliability.md) | At-most-once/at-least-once/exactly-once, idempotent producer + transactions, the dual-write problem, DLQs |
| 5 | [Retention, Compaction & Performance](./Part5-Retention-Compaction-and-Performance.md) | Segment files, retention policies, log compaction, page cache/zero-copy, common production tuning knobs |

## How to use this

- Part 1 is foundational — the storage model (partitions as append-only logs, replication via ISR) is what every later part builds on.
- Every part ends with an **Interview Q&A** section.
- Cross-references: `Multhithreading/` (consumer poll loops and thread-per-consumer patterns), `Microservice-Patterns/` (Kafka as the backbone for event-driven architectures and the Saga pattern), `SpringAndSpringBoot/` (Spring Kafka wraps everything here behind `@KafkaListener`/`KafkaTemplate`).

---

*Last updated: July 2026*
