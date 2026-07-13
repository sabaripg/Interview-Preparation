# Multithreading — Interview Preparation

A complete, corrected, and expanded multithreading interview guide — basic fundamentals through advanced concurrency, each part with worked code examples, expected outputs, pitfalls, and interview Q&A.

## 📚 Contents

| # | Part | Covers |
|---|---|---|
| 1 | [Fundamentals](./Part1-Fundamentals.md) | Program/Process/Thread, thread creation, lifecycle states, monitor lock, wait/notify, priority, daemon threads, interrupting threads |
| 2 | [Executors & Thread Pools](./Part2-Executors-ThreadPools.md) | ExecutorService, ThreadPoolExecutor internals, pool types, Callable/Future, rejection policies |
| 3 | [Coordination Utilities](./Part3-Coordination-Utilities.md) | CountDownLatch, CyclicBarrier, Phaser, Semaphore, Exchanger, Condition, missed signals |
| 4 | [Locks](./Part4-Locks.md) | synchronized vs ReentrantLock, situational lock-selection guide, ReadWriteLock, StampedLock, optimistic vs pessimistic locking, deadlock/livelock/starvation |
| 5 | [Memory Model & Lock-Free](./Part5-Memory-Model-LockFree.md) | volatile, visibility vs atomicity, false sharing, CAS, ABA problem, LongAdder, JMM happens-before |
| 6 | [Concurrent Collections](./Part6-Concurrent-Collections.md) | ConcurrentHashMap internals, CopyOnWriteArrayList, BlockingQueue variants, ThreadLocal, LRU cache design |
| 7 | [Fork/Join & Parallelism](./Part7-ForkJoin-Parallelism.md) | Concurrency vs parallelism, ForkJoinPool, RecursiveTask, work-stealing |
| 8 | [Virtual Threads & Exceptions](./Part8-Virtual-Threads-Exceptions.md) | Virtual vs platform threads, uncaught exception handling |
| — | [CompletableFuture — Complete Guide](./CompletableFuture-Complete-Guide.md) | Basic → advanced, with a quick "which method do I need" reference table |

## How to use this

- Start at Part 1 and work forward if you're building fundamentals up.
- Jump straight to a part if you're revising a specific topic before an interview.
- Every part ends with an **Interview Q&A** section — good for a fast pre-interview pass.
- Diagrams referenced inline (`images/*.svg`) sit one folder up from these files.

---

*Last updated: July 2026*
