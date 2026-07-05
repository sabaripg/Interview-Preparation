# Part 8 — Virtual Threads & Uncaught Exception Handling

> Virtual threads vs platform threads, what happens when a thread dies from an uncaught exception. Interview Q&A at the end.

## Virtual Threads vs Platform Threads

**Platform thread — what it is:** backed 1-to-1 by a real OS thread. Creating one is relatively expensive (a real kernel-level system call), and the OS scheduler manages it directly. A thread blocked waiting on I/O (DB call, HTTP call) ties up that OS thread doing nothing for the entire wait — a major source of resource underutilization in traditional thread-per-request server architectures.

```
Before Java 21:
Java Thread
     │
     ▼
Operating System Thread

After Java 21:
Virtual Thread
      │
      ▼
JVM Scheduler
      │
      ▼
Platform (Carrier) Thread
      │
      ▼
Operating System Thread
```

**Virtual thread (Java 21+) — what it does:** a lightweight thread managed and scheduled by the *JVM* instead of the OS, cheap enough to create millions of. Many virtual threads share a small pool of underlying platform ("carrier") threads — when a virtual thread blocks on I/O, the JVM unmounts it from its carrier thread and runs a different virtual thread on that same carrier, instead of leaving an OS thread idle.

**Why it's called "virtual":** it's not permanently tied to a specific OS thread — a *logical* thread the JVM creates. The JVM decides when it runs, where it runs, and which carrier thread executes it — just like a virtual machine isn't a physical machine, a virtual thread isn't a dedicated OS thread.

```java
// Platform thread
Thread platform = Thread.ofPlatform().start(() -> System.out.println("platform thread: " + Thread.currentThread()));

// Virtual thread
Thread virtual = Thread.ofVirtual().start(() -> System.out.println("virtual thread: " + Thread.currentThread()));
```
**Output:**
```
platform thread: Thread[#21,Thread-0,5,main]
virtual thread: VirtualThread[#22]/runnable@ForkJoinPool-1-worker-1
```

**Why it matters:** virtual threads were introduced to make traditional blocking code scalable — especially for I/O-bound applications (DB calls, REST calls, file/network I/O) — **without** requiring developers to rewrite applications using reactive programming. A traditional blocking-style thread-per-request programming model (simple, easy to reason about, unlike reactive/WebFlux) can scale to handling huge numbers of concurrent connections cheaply.

> ⚠️ **Pitfall — the strongest way to demonstrate real understanding:** virtual threads target the exact same problem WebFlux/reactive programming solves, but via a completely different approach — keep the blocking programming model and make the *threads* cheap, rather than making the *code* non-blocking. Naming this contrast explicitly (rather than describing virtual threads in isolation) is what separates a surface answer from a strong one.

## Uncaught Exception Handling

**What happens:** an uncaught exception in a thread terminates **only that thread**. Other threads continue running normally, because each thread has its own independent call stack.

By default, the JVM: prints the exception stack trace to `System.err`, terminates the affected thread, and does **not** stop the entire JVM (unless it's the `main` thread and no other user threads remain).

```java
Thread t1 = new Thread(() -> {
    throw new RuntimeException("Thread-1 failed");
});
Thread t2 = new Thread(() -> {
    while (true) {
        System.out.println("Thread-2 running...");
    }
});
t1.start();
t2.start();
```
**Output:**
```
Exception in thread "Thread-0"
java.lang.RuntimeException: Thread-1 failed

Thread-2 running...
Thread-2 running...
Thread-2 running...
```
Thread-1 dies; Thread-2 continues running unaffected.

**Handling uncaught exceptions — register a handler:**
```java
// JVM-wide default handler
Thread.setDefaultUncaughtExceptionHandler((thread, ex) -> {
    System.err.println(thread.getName() + " died: " + ex.getMessage());
});

// Per-thread handler
Thread t = new Thread(task);
t.setUncaughtExceptionHandler((thread, ex) -> {
    // log or alert
});
```

`Thread.setDefaultUncaughtExceptionHandler()` (JVM-wide) or `setUncaughtExceptionHandler()` (per-thread) lets you observe and react to thread deaths — critical for any long-running worker/consumer thread, where a silently-dead thread otherwise just looks like "stopped processing" with no obvious cause in the logs.

> ⚠️ **Pitfall:** this is exactly why `ExecutorService.submit()` is preferable to raw `Thread` + `execute()` for anything you care about failing loudly (Part 2) — `submit()` captures the exception in the returned `Future`, surfaced on `.get()`, rather than relying on an uncaught-exception handler you might forget to wire up. A pool worker thread dying from an uncaught exception in a badly-configured thread pool can also silently shrink your effective pool size over time if the pool doesn't replace it — worth naming as a production symptom to watch for.

---

## Interview Q&A

**Q: Virtual threads vs platform threads — what problem do virtual threads actually solve?**
Covered above.

**Q: What happens if a thread throws an uncaught exception — does it take down other threads?**
Covered above.
