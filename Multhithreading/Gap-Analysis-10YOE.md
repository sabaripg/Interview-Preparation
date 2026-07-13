# Multithreading Notes — Gap Analysis for a 10-Year-Experience Interview

> Reviewed: README + Part 1–8 + CompletableFuture guide. The existing material is excellent for fundamentals through upper-mid-level (JMM, locks, executors, CompletableFuture, virtual threads are all covered with real pitfalls, not just definitions). At 10 years of experience — especially with your Spring Boot / Kafka / Microservices / AWS background — interviewers stop asking "what is a deadlock" and start asking how you've used, tuned, and debugged concurrency in production systems. That's where the current notes have gaps.

## 1. Production diagnosis — underdeveloped, not missing

Part 4 has a good `jstack`/`Thread.holdsLock()` section, but a 10 YOE candidate is expected to go further:

- Reading a full thread dump under load: recognizing `BLOCKED` vs `WAITING` threads at scale, spotting thread pool exhaustion (hundreds of threads stuck at the same stack frame), and identifying a stuck connection pool vs. a stuck downstream call.
- Async-profiler / JFR (Java Flight Recorder) for CPU and lock-contention profiling — `jstack` only gives point-in-time snapshots; flame graphs show where time is actually going.
- A concrete "war story" structure: symptom (latency spike / thread pool exhaustion) → tool used → root cause → fix. Interviewers at this level often ask "tell me about a concurrency bug you fixed in production" — the notes have no scaffolding for this kind of answer.

## 2. Thread pool sizing — formula is missing

Part 2 explains *how* `ThreadPoolExecutor` fills up, but never gives the sizing formula senior engineers are expected to reason from:

```
N_threads = N_cpu × U_target × (1 + W/C)
```
(CPU count × target utilization × (1 + wait-time/compute-time ratio)) — i.e., CPU-bound work needs a pool close to core count; I/O-bound work needs a much larger pool because threads spend most of their time waiting, not computing. Missing entirely: how this reasoning applies to sizing Tomcat's thread pool, an `@Async` executor, or a Kafka consumer thread count.

## 3. Spring-specific concurrency — missing (directly relevant to your resume)

Nothing on how concurrency shows up day-to-day in a Spring Boot codebase:

- `@Async` + custom `TaskExecutor`/`ThreadPoolTaskExecutor` bean configuration, and why the default `SimpleAsyncTaskExecutor` (unbounded, one-thread-per-task) is a production trap.
- Tomcat's embedded thread pool (`server.tomcat.threads.max`) and how it interacts with a downstream-call thread pool — a classic cascading-exhaustion scenario.
- `@Transactional` and thread-bound resources: why a DB transaction / Hibernate session is tied to the calling thread, and what breaks if you hand work off to another thread mid-transaction.
- HikariCP pool sizing and the "thread pool bigger than connection pool" starvation trap.

## 4. Kafka consumer concurrency — missing (directly relevant to your resume)

Given your Kafka background, this is a near-certain interview angle:

- `KafkaConsumer` is **not thread-safe** — the single-threaded access model, and why calling `poll()` concurrently from two threads throws `ConcurrentModificationException`.
- Two real multi-threading patterns for consumers: one-consumer-per-partition (thread-per-consumer) vs. single-consumer-thread handing records to a worker pool (with the tradeoff of losing partition-order guarantees in the second approach).
- Consumer group rebalancing and what happens to in-flight work on a thread when a partition is revoked mid-processing.

## 5. Distributed concurrency — missing

Everything in the current notes is single-JVM. At 10 YOE, "concurrency" questions often pivot to distributed systems in a microservices context:

- Distributed locks (Redis/Redisson, ZooKeeper, `SELECT ... FOR UPDATE`) — why a `synchronized` block or `ReentrantLock` does nothing across multiple service instances.
- Idempotency and exactly-once-ish processing when retries + concurrency combine (duplicate message delivery, at-least-once Kafka semantics + concurrent consumers).
- Leader election as a coordination pattern.

## 6. Newer JDK concurrency features — partially missing

Part 8 covers virtual threads well but misses two things that come up in current interviews:

- **Thread pinning**: a virtual thread running a `synchronized` block (pre-JDK 24 behavior) pins its carrier thread for the duration — defeating the scalability benefit if legacy `synchronized` code is on the hot path. This is the single most-asked "gotcha" follow-up to any virtual threads question.
- **Structured Concurrency** (`StructuredTaskScope`) and **Scoped Values** (replacement for `ThreadLocal` in virtual-thread-heavy code) — even in preview, senior candidates are expected to know these exist and why.

## 7. Testing concurrent code — missing

No coverage of how to actually *test* the code these notes describe:

- Why `Thread.sleep()`-based synchronization in tests is flaky, and using `CountDownLatch`/`CyclicBarrier` to deterministically force interleavings instead.
- Tools like `jcstress` for finding memory-model bugs that don't show up on a single run.
- Basic JMH benchmarking mention — senior candidates are sometimes asked to reason about throughput/latency tradeoffs, not just correctness.

## 8. Resiliency patterns — missing

Bulkhead/circuit-breaker (Resilience4j) as thread-isolation strategy — dedicating separate thread pools per downstream dependency so one slow dependency can't exhaust the pool used by everything else. This is a standard "how would you design this" senior-level follow-up to the thread pool material already in Part 2.

---

## Suggested priority if you're short on prep time

1. Thread pool sizing formula + Spring `@Async`/Tomcat/HikariCP interaction (#2, #3) — highest odds of coming up given your background.
2. Kafka consumer thread-safety and concurrency patterns (#4) — same reason.
3. Virtual thread pinning (#6) — cheap to learn, very commonly used as a follow-up gotcha.
4. Production diagnosis war-story framing (#1) — behavioral/technical hybrid, worth having 1-2 real stories ready regardless of what's written down.
5. Distributed locks + testing concurrency (#5, #7) — lower probability but rounds out a senior answer if asked "how do you know your concurrent code is correct."

Items 6 (structured concurrency specifics) and 8 (resiliency) are good-to-have but lowest priority for a Java/Spring/Kafka-focused 10 YOE interview.
