# Multithreading — Complete Series
 
Consolidated from Parts 1–5 + Questions. GitHub auto-links headings, so each **"Topics covered"** list below is clickable and jumps straight to that section.
 
## 📑 Series Navigation
- [Part 1 — Basics](#part-1--basics)
- [Part 2 — Thread Pools](#part-2--thread-pools)
- [Part 3 — CountDownLatch](#part-3--countdownlatch)
- [Part 4 — Locks](#part-4--locks)
- [Part 5 — Lock-Free Concurrency](#part-5--lock-free-concurrency)
- [Questions](#questions)
---
 
# Part 1 — Basics
 
**Topics covered:**
- [Process and Thread](#process-and-thread)
- [Thread Creation Ways](#thread-creation-ways)
- [Thread Lifecycle States](#thread-lifecycle-states)
- [Monitor Lock](#monitor-lock)
- [Producer-Consumer using wait()/notifyAll()](#producer-consumer-using-waitnotifyall)
- [wait() vs notify() vs notifyAll()](#wait-vs-notify-vs-notifyall)
- [Thread Priority](#thread-priority)
- [Daemon Thread](#daemon-thread)
## Process and Thread
 
In Java, a **process** is an executing instance of a Java application. When you run a `.java` program via the JVM, the OS creates a process and allocates resources — heap, stack, method area, CPU time, file handles. Each process runs in its own memory space, independent of other processes.
 
**Example:** two separate `.java` programs run as two separate OS processes, each with its own JVM instance.
 
A JVM process gets memory split as:
- **Heap** — objects
- **Stack** — method calls, local vars, references
- **Method Area / Metaspace** — class metadata, statics
- **Native Memory** — JVM internals, threads, direct buffers
A **thread** is a lightweight process — the smallest independently schedulable sequence of instructions. One process starts with a single **main thread**, from which more threads can be spawned.
 
```java
public class MultithreadingLearning {
    public static void main(String[] args) {
        System.out.println("Thread Name: " + Thread.currentThread().getName());
    }
}
// Output: Thread Name: main
```
 
**Benefits** of multithreading: task parallelism, responsiveness, resource sharing.
**Challenges:** deadlocks/race conditions, synchronization overhead, harder debugging.
 
**Context switching** — the CPU pausing one thread (saving state) and resuming another. Done by the **OS scheduler**, not the JVM itself, since Java threads map to native OS threads.
 
## Thread Creation Ways
 
Two ways: **implement Runnable** (preferred) or **extend Thread** (less flexible — uses up your one shot at single inheritance).
 
```java
// Way 1 — Runnable
public class MultithreadingLearning implements Runnable {
    public void run() {
        System.out.println("Code executed by: " + Thread.currentThread().getName());
    }
}
Thread thread = new Thread(new MultithreadingLearning());
thread.start();
```
 
```java
// Way 2 — extends Thread
public class MultithreadingLearning extends Thread {
    public void run() {
        System.out.println("Code executed by: " + Thread.currentThread().getName());
    }
}
new MultithreadingLearning().start();
```
 
> 💡 **Interview gold:** `start()` creates a new OS-level call stack and internally invokes `run()` on the new thread. Calling `run()` directly executes on the **current** thread — no new thread is created. Thread state transitions are managed by the Scheduler + JVM.
 
> ⚠️ **Pitfall:** `stop()`, `suspend()`, `resume()` are **deprecated** — unsafe, can leave shared objects in an inconsistent state or cause permanent deadlock (suspend holds locks while paused).
 
## Thread Lifecycle States
 
**New** — Created but not started. Just an object in memory.
 
**Runnable** — Ready to run, waiting for CPU time.
 
**Running** — Actively executing.
 
**Blocked** — Waiting to acquire a lock held by another thread, or blocked on I/O. Releases all monitor locks it holds.
 
**Waiting** — After `wait()`/`join()` (no timeout) or `LockSupport.park()`. Releases the monitor lock.
 
**Timed Waiting** — Via `sleep(ms)`, `wait(ms)`, `join(ms)`. **Key distinction: `sleep(ms)` does NOT release the lock; `wait(ms)` DOES.**
 
**Terminated** — Finished executing; cannot be restarted.
 
## Monitor Lock
 
A monitor lock ensures only one thread at a time executes a `synchronized` block/method on a given object.
 
```java
public class MonitorLockExample {
    public synchronized void task1() {
        System.out.println("Inside task1");
        Thread.sleep(1000);
    }
    public void task2() {
        System.out.println("task2, before synchronized");
        synchronized (this) { System.out.println("task2, inside synchronized"); }
    }
    public void task3() { System.out.println("task3"); }
}
```
 
> 💡 Only the code **inside** a `synchronized` block waits on the lock. Unsynchronized code (like `task3()`) can run anytime, regardless of who holds the lock.
 
## Producer-Consumer using wait()/notifyAll()
 
> 💡 This is a **simplified single-item** version (boolean flag) — not a true bounded buffer. Part 2 covers the real bounded-queue version with `BlockingQueue`.
 
```java
class SharedResource {
    private boolean isItemPresent = false;
    public synchronized void addItem() {
        isItemPresent = true;
        notifyAll();
    }
    public synchronized void consumeItem() {
        while (!isItemPresent) {
            try { wait(); } catch (InterruptedException e) {}
        }
        isItemPresent = false;
    }
}
```
 
> ⚠️ **Why `while` not `if`?** (1) **Spurious wakeups** — JVM spec allows a thread to wake without a real `notify()`. (2) With `notifyAll()` and multiple consumers, more than one wakes but only one should proceed — re-checking prevents consuming a non-existent item.
 
## wait() vs notify() vs notifyAll()
 
All three live on `Object` (not `Thread`) and require holding the object's monitor (i.e. inside a `synchronized` block), or they throw `IllegalMonitorStateException`.
 
- **wait()** — releases the lock, moves to WAITING/TIMED_WAITING.
- **notify()** — wakes one *arbitrary* waiting thread; it must re-acquire the lock before proceeding.
- **notifyAll()** — wakes all waiting threads; only one at a time actually re-acquires the lock.
> ⚠️ **Prefer `notifyAll()` over `notify()`** in real code — `notify()` can wake the "wrong" thread when multiple threads wait for different conditions on the same object, causing missed signals.
 
## Thread Priority
 
A **hint** (1–10, default 5 via `Thread.NORM_PRIORITY`) to the scheduler — not a guarantee, and behavior is OS-dependent.
 
> ⚠️ Don't design real concurrency logic around thread priority.
 
## Daemon Thread
 
A background/service thread (e.g. Garbage Collector, Finalizer, Signal dispatcher). The JVM does **not** wait for daemon threads before exiting.
 
> ⚠️ **Must call `setDaemon(true)` before `start()`** — calling it after throws `IllegalThreadStateException`.
 
---
 
# Part 2 — Thread Pools
 
**Topics covered:**
- [Bounded-Buffer Producer-Consumer](#bounded-buffer-producer-consumer)
- [Why Thread Pools?](#why-thread-pools)
- [ThreadPoolExecutor](#threadpoolexecutor)
- [Blocking Queue](#blocking-queue)
- [Types of Thread Pool Executors](#types-of-thread-pool-executors)
## Bounded-Buffer Producer-Consumer
 
Classic problem: producer/consumer share a **fixed-size** queue. Producer must wait if the buffer is full; consumer must wait if it's empty.
 
```java
public class SharedResource {
    Queue<Integer> queue = new LinkedList<>();
    int bufferSize;
    SharedResource(int bufferSize) { this.bufferSize = bufferSize; }
 
    public synchronized void producer(int i) {
        while (queue.size() == bufferSize) {
            try { wait(); } catch (Exception e) { e.printStackTrace(); }
        }
        queue.add(i);
        notifyAll();
    }
 
    public synchronized void consumer() {
        while (queue.size() == 0) {
            try { wait(); } catch (Exception e) { e.printStackTrace(); }
        }
        System.out.println("data consumed: " + queue.remove());
        notify();
    }
}
```
 
> 📷 *Console output screenshot from the original post: see [Part 2 on Medium](https://medium.com/@sabaripinterview/multithreading-part-2-a755b5ff1b0e) if you want to re-embed that image here.*
 
## Why Thread Pools?
 
**Problem without a pool:** creating a thread costs real OS resources; 1000 requests → 1000 threads → massive memory + wasted CPU on context switching.
 
**Solution:** pre-create N threads that wait for work, reuse them, cap max concurrency, and queue tasks when all threads are busy.
 
**Benefits:** reduced latency, bounded resource usage, higher throughput, task queuing instead of failure.
 
## ThreadPoolExecutor
 
```java
ExecutorService executor = Executors.newFixedThreadPool(3);
for (int i = 1; i <= 5; i++) {
    final int taskId = i;
    executor.submit(() ->
        System.out.println("Task " + taskId + " by " + Thread.currentThread().getName()));
}
executor.shutdown();
```
 
```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    2,                              // corePoolSize
    5,                              // maximumPoolSize
    60, TimeUnit.SECONDS,           // keepAliveTime
    new LinkedBlockingQueue<>(10),  // workQueue
    new ThreadPoolExecutor.AbortPolicy() // rejection handler
);
```
 
- **corePoolSize** — min threads kept alive.
- **maximumPoolSize** — max threads including core.
- **keepAliveTime** — how long idle non-core threads wait before dying.
- **workQueue** — where pending tasks wait.
- **handler** — rejection strategy once queue + threads are both full.
## Blocking Queue
 
Holds tasks before a worker thread picks them up.
 
- **Bounded queue** — fixed capacity, e.g. `ArrayBlockingQueue`.
- **Unbounded queue** — no fixed capacity, e.g. `LinkedBlockingQueue`.
## Types of Thread Pool Executors
 
### 1. Fixed Thread Pool
```java
ExecutorService executor = Executors.newFixedThreadPool(5);
```
Fixed worker count; extra tasks queue. Best for web servers, DB ops, microservices with bounded concurrency. **Con:** tasks wait if all threads are busy.
 
### 2. Cached Thread Pool
```java
ExecutorService executor = Executors.newCachedThreadPool();
```
No fixed upper limit — creates new threads on demand, reuses idle ones, kills idle threads after 60s. Great for many short-lived async tasks.
 
> ⚠️ **Risk:** can create unbounded threads under heavy load → resource exhaustion / OOM.
 
### 3. Scheduled Thread Pool
```java
ScheduledExecutorService executor = Executors.newScheduledThreadPool(10);
executor.schedule(() -> System.out.println("Task"), 5, TimeUnit.SECONDS);
executor.scheduleAtFixedRate(() -> System.out.println("Running"), 0, 10, TimeUnit.SECONDS);
```
Supports delayed and recurring tasks.
 
### 4. Single Thread Executor
```java
ExecutorService executor = Executors.newSingleThreadExecutor();
```
Exactly one worker thread; tasks execute strictly in submission order.
 
### 5. Work-Stealing Thread Pool
```java
ExecutorService executor = Executors.newWorkStealingPool();
```
Introduced Java 8, built on `ForkJoinPool`. Idle threads "steal" tasks from busy threads' queues to maximize CPU utilization.
 
> 💡 Best for recursive/divide-and-conquer algorithms and parallel streams. **Not** ideal for blocking I/O tasks.
 
---
 
# Part 3 — CountDownLatch
 
**Topics covered:**
- [What is CountDownLatch?](#what-is-countdownlatch)
- [Key Features](#key-features)
- [When to Use](#when-to-use)
- [Basic Usage Example](#basic-usage-example)
- [Advantages / Disadvantages](#advantages--disadvantages)
- [Waiting with Timeout](#waiting-with-timeout)
## What is CountDownLatch?
 
`CountDownLatch` (from `java.util.concurrent`, Java 5+) synchronizes one or more threads by making them wait for a count of events to reach zero before proceeding.
 
## Key Features
 
- Initialized with a fixed count.
- Each event decrements the count via `countDown()`.
- Waiting threads block on `await()` until count reaches zero.
- **One-time use** — cannot be reset.
## When to Use
 
- Starting multiple threads simultaneously.
- Waiting for several threads to finish before proceeding.
- Splitting a task into subtasks and waiting on all of them.
- Simulating coordinated scenarios in tests.
## Basic Usage Example
 
```java
CountDownLatch latch = new CountDownLatch(threadCount);
WorkerThread thread = new WorkerThread(latch);
for (int i = 0; i <= threadCount; i++) {
    Thread t = new Thread(thread);
    t.start();
}
System.out.println("Main waiting...");
latch.await();
System.out.println("All operations done, resuming.");
 
class WorkerThread implements Runnable {
    CountDownLatch latch;
    public void run() {
        try {
            Thread.sleep(5000);
        } finally {
            latch.countDown();
        }
    }
}
```
 
> 📷 *Console output screenshot: see [Part 3 on Medium](https://medium.com/@sabaripinterview/multithreading-part-3-03980c5bdd04) if you want to re-embed it here.*
 
## Advantages / Disadvantages
 
**Advantages:** simplifies thread coordination, improves performance by only proceeding when needed, more readable than raw wait/notify.
 
**Disadvantages:** one-time use only (can't reset), incorrect usage risks deadlock, less flexible than `CyclicBarrier`/`Semaphore`.
 
## Waiting with Timeout
 
```java
if (latch.await(5, TimeUnit.SECONDS)) {
    System.out.println("All workers finished within timeout.");
} else {
    System.out.println("Timeout reached before all workers finished.");
}
```
 
---
 
# Part 4 — Locks
 
**Topics covered:**
- [ReentrantLock](#reentrantlock)
- [ReadWriteLock](#readwritelock)
- [StampedLock](#stampedlock)
- [Semaphore](#semaphore)
- [Condition](#condition)
## ReentrantLock
 
`ReentrantLock` (`java.util.concurrent.locks`) gives **explicit locking** with more control than `synchronized`.
 
**1. Try-lock (avoid indefinite blocking)**
```java
if (lock.tryLock()) {
    try { /* critical section */ } finally { lock.unlock(); }
}
```
 
**2. Lock with timeout**
```java
if (lock.tryLock(2, TimeUnit.SECONDS)) { /* got lock */ }
```
 
**3. Fairness policy**
```java
Lock fair = new ReentrantLock(true);   // first-come-first-served
Lock unfair = new ReentrantLock();     // default, higher throughput
```
 
**4. Interruptible locking**
```java
lock.lockInterruptibly(); // possible to interrupt while waiting — NOT possible with synchronized
```
 
**5. Multiple condition variables**
```java
Condition notFull = lock.newCondition();
Condition notEmpty = lock.newCondition();
```
 
> 💡 Fine-grained coordination beyond a single `wait()`/`notify()` pair per object.
 
## ReadWriteLock
 
Separates locks for reading and writing: **Read Lock** = shared (multiple readers OK), **Write Lock** = exclusive.
 
> 💡 **Exact rules:**
> - ✅ Multiple readers together — allowed
> - ❌ Reader + Writer together — not allowed
> - ❌ Multiple writers together — not allowed
> - ✅ Writer alone — allowed
 
**Why:** in most real systems reads are frequent and writes are rare — plain `synchronized`/`ReentrantLock` only lets one reader at a time, hurting throughput. `ReadWriteLock` allows concurrent reads.
 
## StampedLock
 
Introduced in Java 8, provides higher-performance locking for read-heavy workloads via a **stamp** (long value) returned on lock acquisition, used later to unlock/validate.
 
Three modes: **Write Lock** (exclusive), **Read Lock** (shared), **Optimistic Read** (lock-free, fastest).
 
> ⚠️ **Why not just ReadWriteLock?** Under sustained read load a writer can starve indefinitely (never a gap with zero readers). Also, every reader still pays CAS overhead on the shared reader-count, even without blocking.
 
**Optimistic Read — how it actually works:**
1. `tryOptimisticRead()` returns a stamp (version number) — **no lock, no CAS, no blocking of writers at all**.
2. Reader copies every needed field into local variables — none of it can be trusted yet, since a writer could be mutating mid-read.
3. Reader calls `validate(stamp)`. If valid (no write occurred since), the locally-copied values are safe to use.
4. **If validation fails**, the optimistic read is discarded and the reader falls back to a normal `readLock()` (pessimistic) to redo the read safely.
## Semaphore
 
Controls how many threads can access a shared resource **simultaneously**, via a pool of permits (`java.util.concurrent.Semaphore`) — think of it as N entry passes rather than a single lock.
 
**Use cases:** only 10 DB connections, only 5 concurrent API calls, only 3 threads in a critical section.
 
> 💡 `Lock` = 1 thread only. `Semaphore` = N threads.
 
## Condition
 
`Condition` (`java.util.concurrent.locks`) is the modern replacement for `wait()`/`notify()`, used together with a `Lock` (typically `ReentrantLock`).
 
```java
Lock lock = new ReentrantLock();
Condition condition = lock.newCondition();
boolean ready = false;
 
void waitForSignal() {
    lock.lock();
    try {
        while (!ready) condition.await();
    } finally { lock.unlock(); }
}
void sendSignal() {
    lock.lock();
    try { ready = true; condition.signal(); } finally { lock.unlock(); }
}
```
 
**`signal()`** wakes exactly one waiting thread. **`signalAll()`** wakes all of them.
 
---
 
# Part 5 — Lock-Free Concurrency
 
**Topics covered:**
- [Lock-Free Mechanism](#lock-free-mechanism)
- [The "Not Atomic" counter++ Problem](#the-not-atomic-counter-problem)
- [CAS Logic and Implementation](#cas-logic-and-implementation)
- [The ABA Problem](#the-aba-problem)
- [Volatile vs CAS](#volatile-vs-cas)
## Lock-Free Mechanism
 
A **lock-free mechanism** lets multiple threads work on shared data **without locks** (`synchronized`/`ReentrantLock`) — relying instead on atomic operations like **CAS (Compare-And-Swap)**, guaranteeing at least one thread always makes progress, avoiding deadlocks.
 
**CAS involves 3 parameters:** memory location, expected value, and new value (written only if current value still matches expected).
 
## The "Not Atomic" counter++ Problem
 
`counter++` is really 3 steps: **Load** → **Increment** → **Assign back**. Because these are separate, a race condition can occur if another thread modifies the value between Load and Assign.
 
| Time | Thread A | Thread B | Counter in memory |
|---|---|---|---|
| T1 | Load: reads 10 | — | 10 |
| T2 | — | Load: reads 10 | 10 |
| T3 | Increment: 10+1=11 | — | 10 |
| T4 | Assign: writes 11 | — | 11 |
| T5 | — | Increment: 10+1=11 | 11 |
| T6 | — | Assign: writes 11 **(data lost!)** | 11 |
 
> ⚠️ **Result:** one increment is completely overwritten — expected 12, got 11.
 
```java
// Not thread-safe
class SharedResource {
    int counter = 0;
    public void increment() { counter++; } // race condition
}
 
// Fix 1 — synchronized (blocks, mutual exclusion)
class SharedResourceSync {
    private int counter = 0;
    public synchronized void increment() { counter++; }
}
 
// Fix 2 — lock-free via AtomicInteger (non-blocking, uses CAS)
class SharedResourceAtomic {
    private AtomicInteger counter = new AtomicInteger(0);
    public void increment() { counter.incrementAndGet(); }
}
```
 
> 💡 **AtomicInteger** is more scalable under high contention since it's non-blocking, vs. `synchronized` which works correctly but blocks threads (fine under low contention).
 
## CAS Logic and Implementation
 
CAS is typically implemented as a **Read-Modify-Update loop** that retries until it succeeds:
 
```java
do {
    expectedValue = read(memoryLocation);
    newValue = expectedValue + 1;
} while (!CAS(memoryLocation, expectedValue, newValue));
```
 
The processor: **Read** current value → **Compare** to expected → **Swap** if they match, else retry.
 
## The ABA Problem
 
1. Thread 1 reads value A.
2. Thread 1 is preempted (paused).
3. Thread 2 changes A → B → back to A.
4. Thread 1 resumes, does CAS, sees A, assumes nothing changed — swap succeeds.
> ⚠️ **Danger:** the value looks unchanged, but system *state* may have changed (e.g. in a linked list, a node could've been deleted and a different node inserted with the same value). Solved using a version number or timestamp alongside the value.
 
## Volatile vs CAS
 
**Volatile** = a **visibility** guarantee (a change is immediately seen by other threads). **CAS** = an **atomicity** guarantee (the whole read-modify-write cycle is one unbreakable event).
 
> ⚠️ **Crucial rule:** `volatile` only makes *single* reads/writes atomic. It does **not** make compound operations like `count = count + 1` atomic.
 
| Time | Thread A | Thread B | RAM | Why it fails |
|---|---|---|---|---|
| T1 | Reads 10 | — | 10 | volatile ensures visibility of latest value |
| T2 | — | Reads 10 | 10 | Both threads now hold local copy = 10 |
| T3 | Calculates 11 | Calculates 11 | 10 | Calculation happens in CPU registers, not RAM |
| T4 | Writes 11 | — | 11 | RAM updated by Thread A |
| T5 | — | Writes 11 | 11 | ❌ Lost update — should be 12 |
 
> 💡 **Interview one-liner:** `AtomicInteger.incrementAndGet()` uses CAS internally, guaranteeing atomicity + visibility while avoiding the blocking overhead of traditional locks.
 
---
 
# Questions
 
**Topics covered:**
- [ThreadLocal](#threadlocal--real-example)
- [ReentrantLock vs synchronized](#reentrantlock-vs-synchronized-1)
- [ForkJoinPool / RecursiveTask](#forkjoinpool-and-recursivetask)
- [Phaser](#phaser)
- [CyclicBarrier](#cyclicbarrier)
- [Callable vs Runnable](#callable-vs-runnable)
- [Future vs CompletableFuture](#future-vs-completablefuture)
- [Class-level Locks](#class-level-locks)
- [thenApply / thenAccept / thenRun](#thenapply-vs-thenaccept-vs-thenrun-completablefuture)
- [Virtual Threads vs Platform Threads](#virtual-threads-vs-platform-threads)
- [Thread Pool Q&A Set](#thread-pool-qa-set)
## ThreadLocal — real example
 
**Q:** Can you provide an example where ThreadLocal was used effectively?
 
**Real-time example:** in a web app, each user session needs its own auth tokens/preferences. `ThreadLocal` ensures the thread handling a given request sees only its own session data, with no interference from other threads.
 
```java
private static final ThreadLocal<Integer> threadLocalValue = ThreadLocal.withInitial(() -> 1);
public int getValue() { return threadLocalValue.get(); }
public void setValue(int value) { threadLocalValue.set(value); }
```
 
## ReentrantLock vs synchronized
 
ReentrantLock adds: **tryLock()** (non-blocking attempt), **interruptible waiting**, **tryLock(timeout)**, and a **fairness policy** — none of which plain `synchronized` supports.
 
```java
ReentrantLock lock = new ReentrantLock();
lock.lock();
try { /* critical section */ } finally { lock.unlock(); }
```
 
## ForkJoinPool and RecursiveTask
 
**ForkJoinPool** — parallel processing by splitting tasks into smaller pieces (fork/join framework). **RecursiveTask** — a `ForkJoinTask` subclass that returns a result.
 
```java
class FibonacciTask extends RecursiveTask<Integer> {
    private final int n;
    protected Integer compute() {
        if (n <= 1) return n;
        FibonacciTask f1 = new FibonacciTask(n - 1);
        f1.fork();
        FibonacciTask f2 = new FibonacciTask(n - 2);
        return f2.compute() + f1.join();
    }
}
```
 
**How work-stealing works:** each worker has its own task queue; an idle worker "steals" a task from a busy worker's queue — maximizing CPU utilization on multi-core systems.
 
## Phaser
 
A more flexible synchronization barrier than `CountDownLatch`/`CyclicBarrier` — supports **dynamic registration** of parties for complex multi-phase coordination.
 
```java
Phaser phaser = new Phaser(1); // register self
phaser.arriveAndAwaitAdvance(); // wait for all registered parties
phaser.arriveAndDeregister();
```
 
## CyclicBarrier
 
Makes a group of threads wait for each other at a common point — used when several threads compute sub-results that must be combined afterward. Unlike `CountDownLatch`, it's **reusable** (cyclic).
 
```java
CyclicBarrier barrier = new CyclicBarrier(numberOfThreads);
public void run() {
    // computation
    barrier.await();
}
```
 
> 💡 **CountDownLatch vs CyclicBarrier:** CountDownLatch is one-time-use and typically has one/few threads waiting on N events; CyclicBarrier is reusable and makes N threads wait for *each other*.
 
## Callable vs Runnable
 
| | Runnable | Callable |
|---|---|---|
| Package | `java.lang` (since 1.0) | `java.util.concurrent` (since 1.5) |
| Return value | No | Yes — via `call()` |
| Checked exceptions | Cannot throw | Can throw |
| Method to override | `run()` | `call()` |
 
## Future vs CompletableFuture
 
- **Completion handling:** Future needs blocking `get()`; CompletableFuture offers non-blocking `thenAccept()` etc.
- **Composition:** CompletableFuture supports fluent chaining of async operations; Future doesn't.
- **Explicit completion:** CompletableFuture supports `complete()`/`completeExceptionally()`.
## Class-level Locks
 
Every class has a unique class-level lock. A thread executing a `static synchronized` method needs this lock; once acquired, it can call any static synchronized method on that class until it releases.
 
```java
synchronized (Geek.class) {
    // only one thread across ALL instances can be here
}
```
 
## thenApply vs thenAccept vs thenRun (CompletableFuture)
 
- **thenApply(Function)** — transforms the result, returns a new value.
- **thenAccept(Consumer)** — consumes the result, returns nothing.
- **thenRun(Runnable)** — doesn't even have access to the previous result; just runs after completion.
## Virtual Threads vs Platform Threads
 
**Platform thread** — managed/scheduled by the OS; creating one needs a costly system call (kernel thread).
 
**Virtual thread** — managed/scheduled by the JVM, cheap to create/destroy, mapped onto a small number of platform threads — enabling millions of concurrent tasks with a much smaller footprint.
 
> 💡 **Why it matters:** in a typical request → external API/DB call → response flow, the thread spends most of its life just *waiting* for I/O — a huge underutilization of OS threads. Virtual threads make that wait nearly free.
 
## Thread Pool Q&A Set
 
**Why thread pools?** Thread creation/destruction is expensive; pools reuse threads, cap max threads (preventing exhaustion), and improve throughput.
 
**newFixedThreadPool vs newCachedThreadPool:** Fixed = bounded threads + queue, good for CPU-bound work. Cached = threads created on demand with no queue, good for I/O-bound short-lived tasks — but risks unbounded thread growth.
 
**When queue AND threads are both full:** governed by the **Rejection Policy**:
- **AbortPolicy** (default) — throws `RejectedExecutionException`.
- **CallerRunsPolicy** — the calling thread runs the task itself.
- **DiscardPolicy** — silently drops the task.
- **DiscardOldestPolicy** — evicts the oldest queued task, inserts the new one.
**shutdown() vs shutdownNow():** `shutdown()` = orderly, no new tasks accepted, already-submitted tasks run to completion. `shutdownNow()` = attempts immediate stop, interrupts running tasks, cancels pending ones, returns the unexecuted task list.
 
> 💡 **Best practice:** call `shutdown()`, then `awaitTermination()`.
 
**When to use ScheduledThreadPool:** periodic/delayed tasks — cache refresh, health checks, retry-after-delay jobs.
 
