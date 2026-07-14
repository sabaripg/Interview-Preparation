# Part 6 — Concurrent Collections & Thread-Safety Utilities

> ConcurrentHashMap internals, CopyOnWriteArrayList, BlockingQueue variants, ThreadLocal, designing a thread-safe LRU cache. Interview Q&A at the end.

## ConcurrentHashMap vs Hashtable vs Collections.synchronizedMap

**What ConcurrentHashMap does differently:** `Hashtable` and `Collections.synchronizedMap()` both make a map thread-safe with **one single lock** around the entire map — only one thread can touch the map at all, even for two threads reading completely different keys. `ConcurrentHashMap` locks at a much finer grain, so unrelated operations on different parts of the map run truly in parallel.

| | `HashMap` | `Hashtable` | `Collections.synchronizedMap` | `ConcurrentHashMap` |
|---|---|---|---|---|
| Thread-safe? | No | Yes | Yes | Yes |
| Locking granularity | N/A | Entire map (one lock) | Entire map (one lock) | Bucket-level (fine-grained) |
| Null keys/values | Allowed | Not allowed | Depends on backing map | Not allowed |
| Performance under contention | N/A | Poor | Poor | Good |

**How `ConcurrentHashMap` avoids a single global lock, exactly:**
- **Java 7 and earlier** — **segment locking**: the map was divided into a fixed number of `Segment`s, each an independent hash table with its own lock, so operations on different segments proceeded fully in parallel ("lock striping").
- **Java 8+** — segments were dropped in favor of **per-bin (per-bucket) synchronization**: each bucket's head node is synchronized individually for writes, and reads are largely lock-free using `volatile` reads plus CAS for inserting into an empty bin — finer-grained than segment locking.
- Reads (`get()`) are essentially always lock-free in both versions.

> ⚠️ **Pitfall:** the internal mechanism **changed between Java 7 and 8** — describing only the old segment-locking model as current is a dated answer.

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("a", 1);
map.computeIfAbsent("b", k -> 2);
map.merge("a", 10, Integer::sum);
System.out.println(map);
```
**Output:**
```
{a=11, b=2}
```
> ⚠️ **Pitfall:** `ConcurrentHashMap` guarantees thread-safety for individual operations, but **compound actions across multiple calls** (like "check size, then add if under limit") are still not atomic unless you use its atomic compound methods (`computeIfAbsent`, `merge`, `compute`).

## CopyOnWriteArrayList

**What it does:** every mutation (`add`, `remove`, `set`) creates an entirely **new copy** of the underlying array. Reads never need any locking at all and always see a consistent snapshot, since they operate on an array that's never mutated in place once published.

```java
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
list.add("a");
list.add("b");
for (String s : list) {
    list.add("c"); // safe — iterator works on the old snapshot, no ConcurrentModificationException
}
System.out.println(list);
```
**Output:**
```
[a, b, c, c]
```

**Good fit:** read-heavy, write-rare collections — especially where you need to iterate without `ConcurrentModificationException` risk even if another thread mutates concurrently (e.g. a list of event listeners registered once at startup, only ever iterated).

**Trap:** any workload with frequent writes — every write is O(n) (full array copy), so a write-heavy `CopyOnWriteArrayList` is dramatically worse than `ArrayList` + explicit locking or a `ConcurrentHashMap`-style structure.

> ⚠️ **Pitfall — the detail most likely to trip you up:** iterators reflect a **snapshot at iterator-creation time** — they will never show elements added after the iterator was created. This isn't a bug, it's the entire point of the copy-on-write design — state it as a deliberate guarantee, not a limitation.

## BlockingQueue Variants — ArrayBlockingQueue vs LinkedBlockingQueue vs SynchronousQueue

- **ArrayBlockingQueue** — fixed-capacity, array-backed, a single lock shared by both put/take (optionally fair). Bounded by construction — a good default when you want a hard, predictable memory ceiling and explicit backpressure.
- **LinkedBlockingQueue** — optionally bounded (unbounded if no capacity given — see the risk this creates in Part 2's `newFixedThreadPool` discussion), linked-node backed, uses **separate** put/take locks internally — generally higher throughput than `ArrayBlockingQueue` under contention since producers and consumers don't contend on the same lock.
- **SynchronousQueue** — has **zero** internal capacity — every `put()` must rendezvous directly with a waiting `take()`, a pure handoff with no buffering. Used for direct-handoff semantics (`Executors.newCachedThreadPool()` uses one internally — a task is only queued if a thread is immediately available to take it).

> ⚠️ **Pitfall:** always default to an explicitly **bounded** queue (`ArrayBlockingQueue` or a capacity-limited `LinkedBlockingQueue`) in production thread pools — ties directly back to why `Executors.newFixedThreadPool()`'s hidden unbounded `LinkedBlockingQueue` is a real production risk (Part 2).

## ThreadLocal

**What it does:** gives every thread its own private, independent copy of a variable — even though every thread accesses it through the exact same `ThreadLocal` object, each thread reads/writes only its own copy.

**Real-time example:** in a web app, each request/session needs its own auth token/preferences. `ThreadLocal` ensures the thread handling a given request sees only its own session data.

```java
public class ThreadLocalDemo {
    private static final ThreadLocal<Integer> threadLocalValue = ThreadLocal.withInitial(() -> 1);

    public static void main(String[] args) {
        Runnable task = () -> {
            threadLocalValue.set(threadLocalValue.get() + 10);
            System.out.println(Thread.currentThread().getName() + ": " + threadLocalValue.get());
        };
        new Thread(task).start();
        new Thread(task).start();
    }
}
```
**Output (each thread sees its own independent value):**
```
Thread-0: 11
Thread-1: 11
```

**Real-world use cases:**
- **Request context in web apps** (most common) — logged-in user, tenant ID (multi-tenant apps), correlation/trace ID, locale, auth context — accessible from anywhere in the call stack without passing it as a method parameter.
- **Framework internals** — Spring's `RequestContextHolder`, Spring Security's `SecurityContextHolder`, SLF4J/Logback MDC, Hibernate Session Context all use `ThreadLocal` internally.

> ⚠️ **Pitfall:** in thread-pool-based servers (Tomcat, etc.), threads are **reused** across requests. If you forget to call `threadLocalValue.remove()` at the end of a request, stale data leaks into the next request handled by the same pooled thread — a classic memory-leak / data-leak bug. Always clean up in a `finally` block.

## ThreadLocalRandom

**The problem it solves:** a plain `java.util.Random` instance shared across threads is internally backed by a single `AtomicLong` seed, updated via a CAS loop on every call to `nextInt()`/`nextDouble()`/etc. Under high contention (many threads all generating random numbers concurrently), every thread's CAS retries against that *same* seed, causing exactly the kind of contention/cache-line bouncing `LongAdder` was built to avoid for counters (Part 5) — except here it's baked into `Random` itself with no built-in fix.

**What `ThreadLocalRandom` does:** exactly what the name says — combines `ThreadLocal` (Part 6, above) with a random generator, giving each thread its own independent generator instance with its own independent seed. No shared state, no CAS, no contention at all between threads.

```java
public class ThreadLocalRandomDemo {
    public static void main(String[] args) throws InterruptedException {
        Runnable task = () -> {
            int value = ThreadLocalRandom.current().nextInt(1, 100);
            System.out.println(Thread.currentThread().getName() + " got: " + value);
        };
        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);
        t1.start(); t2.start();
        t1.join(); t2.join();
    }
}
```
**One possible output (each thread draws from its own independent generator):**
```
Thread-0 got: 47
Thread-1 got: 12
```

**The API difference to notice:** you never call `new ThreadLocalRandom()` — the constructor is private. You always go through the static `ThreadLocalRandom.current()`, which hands back the calling thread's own instance (creating it lazily on first use, exactly like `ThreadLocal.withInitial()` does).

> ⚠️ **Pitfall:** the mistake this actually catches in an interview is reaching for `new Random()` shared as a `static` field to "avoid creating objects repeatedly" — that instinct is exactly backwards under concurrency. A shared `Random` is a hidden contention point that scales worse as thread count grows; `ThreadLocalRandom.current()` has no shared state to contend over, and is the correct default in any multi-threaded context. Keep a single shared `Random` only when you specifically need a **reproducible seed** across threads for deterministic tests — `ThreadLocalRandom` explicitly does not support seeding for that reason.

## Designing a Thread-Safe, Bounded LRU Cache

**Core structure:** `LinkedHashMap` in access-order mode, overriding `removeEldestEntry()` to evict once size exceeds capacity — gives LRU eviction for free from the JDK.
```java
LinkedHashMap<K, V> cache = new LinkedHashMap<>(capacity, 0.75f, true) {
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
};
```

**Thread safety:** wrapping every method in a single `synchronized` block (or `ReentrantLock`) is the simplest correct approach.

> ⚠️ **Pitfall — the detail candidates most often miss:** `LinkedHashMap`'s access-order reordering happens on **reads** (`get()`) as well as writes — so even "read-only" cache lookups mutate internal state and need the same lock as writes. You cannot get away with a `ReadWriteLock` here unless you accept giving up true-LRU ordering. Assuming a `ReadWriteLock` is a safe optimization for an LRU cache is exactly the mistake that separates a surface-level answer from one that shows real implementation experience.

For real production scale beyond a single lock's throughput ceiling: `ConcurrentHashMap` doesn't give you LRU ordering for free, so a genuinely concurrent LRU cache typically means either accepting approximate LRU (a CLOCK-based or sampled-eviction approximation, like Redis uses) or reaching for a proven library (Caffeine, Guava Cache) rather than hand-rolling one at scale.

---

## Interview Q&A

**Q: How does ConcurrentHashMap avoid a single global lock?**
Covered above — know the Java 7 segment-locking vs Java 8+ per-bin distinction.

**Q: When is CopyOnWriteArrayList a good fit, and when is it a trap?**
Covered above.

**Q: ArrayBlockingQueue vs LinkedBlockingQueue vs SynchronousQueue — how do you choose?**
Covered above.

**Q: What are use cases of ThreadLocal variables in Java?**
Covered above.

**Q: Why prefer ThreadLocalRandom over a shared Random instance in concurrent code?**
Covered above under "ThreadLocalRandom." A shared `Random` serializes every thread through CAS retries on one internal seed; `ThreadLocalRandom.current()` gives each thread its own uncontended generator instance — strictly better for concurrent code, with the one tradeoff being no reproducible shared seeding.

**Q: Design a thread-safe, bounded LRU cache — what are the key decisions?**
Covered above.
