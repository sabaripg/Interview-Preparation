# Part 1 — Fundamentals

> Program → Process → Thread, thread creation, lifecycle, monitor locks, wait/notify, priority, daemon threads. Interview Q&A at the end.

## Program vs Process vs Thread

**What it is:** three layers, each one level more granular than the last.

```
Program   — a passive executable sitting on disk (java.exe, chrome.exe) — does nothing by itself
   │  double-click / launch
   ▼
Process   — a running instance of that program — has its own heap, open files, native resources
   │  spawns
   ▼
Thread    — the smallest unit of execution WITHIN a process
             each thread: own Program Counter, own stack, own registers
             all threads in a process SHARE: heap, objects, static variables
```

A **process** is an independent running program — Chrome, VS Code, Spotify are each their own process, each with its own Heap, Stack, Code, and Resources, completely isolated from one another. If Chrome crashes, VS Code and Spotify keep running, because every process has its own memory. This is called **process isolation**. Since processes can't directly access each other's memory, communicating between them requires IPC (Inter-Process Communication) — sockets, pipes, shared files.

A **thread** is a lightweight unit of execution *within* a process. Multiple threads in the same process share that process's memory space — no IPC needed, just shared variables (with all the synchronization concerns this guide covers).

The practical tradeoff: process isolation gives you fault containment (one process crashing doesn't take down another) at the cost of heavier context-switching and no easy shared-memory communication. Threads are cheap to create and communicate trivially, at the cost of one bad thread being able to corrupt shared state for the whole process.

> ⚠️ **Pitfall — the exact per-thread vs per-process split:** Program Counter, stack, and registers are **per-thread**. Heap, loaded objects, and static variables are **per-process** (shared across all its threads). This exact split is *why* every synchronization tool in this guide exists at all — shared mutable heap state across independently-scheduled threads is the root cause of every race condition you'll see.

## Thread Creation

Two ways: **implement Runnable** (preferred) or **extend Thread** (less flexible).

```java
// Option A — implement Runnable (preferred)
class MyTask implements Runnable {
    public void run() { System.out.println("running on " + Thread.currentThread().getName()); }
}
Thread t = new Thread(new MyTask());
t.start();

// Option B — extend Thread
class MyThread extends Thread {
    public void run() { System.out.println("running on " + Thread.currentThread().getName()); }
}
new MyThread().start();
```
**Output (either way):**
```
running on Thread-0
```

**Why prefer implementing `Runnable`:** a class can `implement` multiple interfaces but `extend` only one class — if your task class already needs to extend something else, extending `Thread` isn't an option, but implementing `Runnable` always is. It also cleanly separates "the task" (a `Runnable`) from "the thing that runs it" (a `Thread`) — exactly the model `ExecutorService` (Part 2) is built around: you submit `Runnable`/`Callable` tasks, not `Thread` subclasses.

> ⚠️ **Interview gold:** `start()` creates a new OS-level call stack and internally invokes `run()` on the new thread. Calling `run()` directly executes on the **current** thread — no new thread is created, it just runs like a normal method call, single-threaded. This is a very common beginner mistake that still "compiles and works," just defeats the entire purpose.

> ⚠️ **Pitfall:** `stop()`, `suspend()`, `resume()` are **deprecated** — unsafe, can leave shared objects in an inconsistent state, or cause permanent deadlock (`suspend()` holds locks while paused).

## Thread Lifecycle States

**What it is:** every thread moves through a fixed set of states, and the JVM tracks which one it's in.

```
             new Thread(...)
                   |
                   v
               +-------+
               |  NEW  |
               +---+---+
                   | start()
                   v
             +-----------+
       +---->| RUNNABLE  |<----------------------+
       |     +-----+-----+                       |
       |           |                              |
       |  needs a  |  wait()/sleep()/join()       | lock acquired /
       |  monitor  |  WITH a timeout              | notify() received /
       |  lock     v                               | timeout elapsed /
       |     +--------------+                       | join() target done
       |     |TIMED_WAITING |-----------------------+
       |     +--------------+
       |           ^
       |           | wait()/join() with NO timeout
+------+-----+     |
|  BLOCKED   |     |
+------+-----+     |
       |      +-----+-----+
       |      |  WAITING  |
       |      +-----------+
       |  lock becomes free
       +--------------------------> (back to RUNNABLE, competes for the lock)

             RUNNABLE
                |  run() completes normally, OR uncaught exception
                v
          +-------------+
          | TERMINATED  |  (final -- cannot transition anywhere else)
          +-------------+
```

| State | Description |
|---|---|
| **New** | The `Thread` object has been constructed but `start()` hasn't been called yet. No OS-level thread exists — this is purely a Java object sitting in memory. |
| **Runnable** | `start()` has been called. The thread is eligible to run, but "eligible" doesn't mean "currently executing" — the OS scheduler decides which RUNNABLE thread actually gets CPU time. Java doesn't distinguish "ready and waiting for CPU" from "actually running right now" — both report as RUNNABLE. |
| **Blocked** | The thread tried to enter a `synchronized` block but another thread holds that object's monitor lock. Resolution is **passive** — the moment the lock-holder releases it, one BLOCKED thread automatically becomes eligible again, no signal needed from anyone. |
| **Waiting** | The thread voluntarily gave up its lock via `wait()` (no timeout), `join()` (no timeout), or `LockSupport.park()`, and is waiting **indefinitely** for another thread to actively do something — call `notify()`/`notifyAll()`, finish (for `join()`), or `unpark()`. Resolution is **active** — it never resolves on its own. |
| **Timed Waiting** | Same as Waiting, but bounded: `sleep(ms)`, `wait(ms)`, `join(ms)`. Resolves when the timeout elapses **or** the same signal that would end a plain Waiting state occurs first, whichever comes first. |
| **Terminated** | `run()` completed, either normally or via an uncaught exception. A dead end — calling `start()` again throws `IllegalThreadStateException`. |

> ⚠️ **Pitfall — the precise BLOCKED vs WAITING distinction:** "Both just mean the thread is stuck" is the shallow answer. **BLOCKED** is specifically about contending for a monitor lock (resolves automatically once the lock frees up — no explicit signal needed). **WAITING** is about a thread that already *voluntarily gave up* its lock and needs another thread to actively notify it — it doesn't resolve just because a resource becomes free. This maps directly onto `jstack` output: distinguishing the two tells you whether you're looking at lock contention (BLOCKED) or a coordination bug like a missed `notify()` (WAITING).

## Thread Scheduler and Time Slicing

**What it does:** the Thread Scheduler is a JVM/OS component that decides which `RUNNABLE` thread actually gets CPU time next — influenced by thread priority, but ultimately OS-dependent and not guaranteed. **Time slicing** is the specific strategy of giving each runnable thread a small, fixed CPU quantum before preempting it and moving to the next thread round-robin — this is the mechanism that makes single-core "concurrency" actually work, interleaving many threads' progress within one core.

> ⚠️ **Pitfall:** exact scheduling behavior (quantum length, whether it's strictly round-robin) is OS/JVM-dependent — don't rely on it for correctness, only for general throughput expectations.

## Interrupting Threads

**What it does:** `Thread.interrupt()` is how one thread asks another to stop what it's doing. Critically, calling it does **not** forcibly stop anything — it only **sets a boolean interrupt flag** on the target thread. What happens next depends entirely on what that thread is doing when the flag gets set:

- If the target thread is currently blocked in `sleep()`, `wait()`, or `join()`, the JVM wakes it immediately, **clears the interrupt flag**, and throws `InterruptedException` right there.
- If the target thread is just running normal code (not blocked in one of those calls), nothing happens automatically — the flag is simply set, and the thread must **cooperatively** check it (`isInterrupted()`) and decide to stop on its own. Nothing forces it to.

```java
class InterruptExample {
    static void example() throws InterruptedException {
        Thread sleepyThread = new Thread(() -> {
            try {
                System.out.println("I am too sleepy... let me sleep for an hour.");
                Thread.sleep(1000 * 60 * 60);
            } catch (InterruptedException ie) {
                // sleep() already cleared the flag as part of throwing this exception
                System.out.println("Flag right after catching: " + Thread.interrupted() + " " + Thread.currentThread().isInterrupted());

                Thread.currentThread().interrupt(); // re-set the flag ourselves, to demonstrate the two check methods
                System.out.println("Oh, someone woke me up!");

                System.out.println("Flag after re-interrupting: " + Thread.currentThread().isInterrupted() + " " + Thread.interrupted());
            }
        });
        sleepyThread.start();
        System.out.println("About to wake up the sleepy thread...");
        sleepyThread.interrupt();
        System.out.println("Woke up sleepy thread...");
        sleepyThread.join();
    }
}
```
**Output:**
```
I am too sleepy... let me sleep for an hour.
About to wake up the sleepy thread...
Woke up sleepy thread...
Flag right after catching: false false
Oh, someone woke me up!
Flag after re-interrupting: true true
```

**Why the first line prints `false false`:** when `sleep()` is interrupted, the JVM clears the interrupt flag as part of throwing `InterruptedException` — by the time you're inside the `catch` block, the flag is already `false` again, before you've called anything yourself.

**Two ways to check the flag — and they behave differently:**

| | `Thread.interrupted()` | `someThread.isInterrupted()` |
|---|---|---|
| Kind | `static` — always checks the **current** thread | Instance method — checks **whatever thread you call it on** |
| Side effect | **Clears** the flag as it reads it | Just reads it, **no side effect** |

> ⚠️ **Pitfall — order of evaluation matters:** in the last print statement above, `isInterrupted()` runs first (sees `true`, doesn't touch the flag), *then* `Thread.interrupted()` runs (also sees `true`, then clears it) — so both print `true`. Swap the call order and the second call would see the flag already cleared by the first, printing `true false` instead. This is exactly the kind of "trace it through" question that separates a memorized answer from real understanding — the flag is mutable state, and *which* method you call, and in what order, changes what the next call sees.

> ⚠️ **Pitfall — interruption is advisory, not forced:** `interrupt()` can only unblock a thread that's already in an interruptible wait (`sleep`/`wait`/`join`), or set a flag a running thread must **choose** to check. A `while(true) { /* tight loop, no blocking call, never checks isInterrupted() */ }` thread will simply never stop no matter how many times you call `interrupt()` on it. See Part 2's "Thread Interruption in the Executor Framework" for how this cooperative model plays out with `Future.cancel()`.

## Monitor Lock

A monitor lock ensures only one thread at a time executes a `synchronized` block/method on a given object.

```java
public class MonitorLockExample {
    public synchronized void task1() throws InterruptedException {
        System.out.println("Inside task1");
        Thread.sleep(1000);
    }
    public void task2() {
        System.out.println("task2, before synchronized");
        synchronized (this) { System.out.println("task2, inside synchronized"); }
    }
    public void task3() { System.out.println("task3"); }

    public static void main(String[] args) {
        MonitorLockExample obj = new MonitorLockExample();
        new Thread(() -> { try { obj.task1(); } catch (InterruptedException e) {} }).start();
        new Thread(obj::task2).start();
        new Thread(obj::task3).start();
    }
}
```
**One possible output:**
```
Inside task1
task3
task2, before synchronized
task2, inside synchronized
```
> Only the code **inside** a `synchronized` block waits on the lock. Unsynchronized code (like `task3()`) runs anytime, regardless of who holds the lock.

## wait() vs notify() vs notifyAll()

**Why these live on `Object`, not `Thread`:** they're not about "pausing a thread" in the abstract — they're about coordinating around one **shared object** that multiple threads touch. That shared object needs somewhere to remember who's waiting on it — that's its **monitor**. Every object already has one (used by `synchronized`), so Java attaches `wait()`/`notify()`/`notifyAll()` to `Object` itself, letting *any* object become a coordination point.

```java
class Mailbox {
    private String message = null;

    synchronized void waitForMessage() throws InterruptedException {
        while (message == null) {
            System.out.println(Thread.currentThread().getName() + ": no message yet, waiting...");
            wait(); // releases Mailbox's monitor, thread parks here
        }
        System.out.println(Thread.currentThread().getName() + ": got message -> " + message);
    }

    synchronized void sendMessage(String msg) {
        this.message = msg;
        System.out.println(Thread.currentThread().getName() + ": sending message, notifying...");
        notify(); // wakes ONE thread waiting on THIS Mailbox object
    }

    public static void main(String[] args) throws InterruptedException {
        Mailbox mailbox = new Mailbox();
        Thread reader = new Thread(() -> {
            try { mailbox.waitForMessage(); } catch (InterruptedException e) {}
        }, "Reader");
        Thread writer = new Thread(() -> mailbox.sendMessage("Hello!"), "Writer");

        reader.start();
        Thread.sleep(500); // make sure Reader starts waiting first
        writer.start();
    }
}
```
**Output:**
```
Reader: no message yet, waiting...
Writer: sending message, notifying...
Reader: got message -> Hello!
```

- **wait()** — releases the lock, moves to WAITING/TIMED_WAITING.
- **notify()** — wakes one *arbitrary* waiting thread; it must re-acquire the lock before proceeding.
- **notifyAll()** — wakes all waiting threads; only one at a time actually re-acquires the lock.

> ⚠️ **Pitfall — the real answer to "why Object, not Thread":** the JVM's bookkeeping of "who is waiting" is attached to the shared object's monitor, not to any `Thread` object. In the example above, `Mailbox` could be shared by 10 readers and 10 writers, and it would still work correctly, precisely because the coordination point is the object, not any particular thread. Prefer `notifyAll()` over `notify()` in real code — `notify()` can wake the "wrong" thread when multiple threads wait for different conditions on the same object.

## Thread Priority

A **hint** (1–10, default 5 via `Thread.NORM_PRIORITY`) to the scheduler about relative importance — **not** a guarantee, and behavior is entirely OS-dependent; Java explicitly does not guarantee priority translates into actual execution order.

```java
Thread t1 = new Thread(() -> System.out.println("Thread 1 running"));
Thread t2 = new Thread(() -> System.out.println("Thread 2 running"));
t1.setPriority(Thread.MIN_PRIORITY); // 1
t2.setPriority(Thread.MAX_PRIORITY); // 10
t1.start();
t2.start();
```
**One possible output (order not guaranteed):**
```
Thread 2 running
Thread 1 running
```
> ⚠️ **Pitfall:** don't design correctness-dependent logic around thread priority. If you need actual ordering guarantees, use coordination primitives instead — `join()`, latches (Part 3), or explicit locks (Part 4).

## Daemon Thread

A background/service thread (Garbage Collector, Finalizer, Signal dispatcher are canonical examples). The JVM does **not** wait for daemon threads before exiting — the moment every remaining thread is a daemon thread, the JVM shuts down immediately, abandoning any daemon threads mid-execution with no cleanup guarantee.

```java
public class DaemonDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread daemonThread = new Thread(() -> {
            int i = 0;
            while (true) {
                System.out.println("Daemon running: " + i++);
                try { Thread.sleep(300); } catch (InterruptedException e) { return; }
            }
        });
        daemonThread.setDaemon(true); // MUST be set before start()
        daemonThread.start();

        System.out.println("Main thread doing work...");
        Thread.sleep(1000);
        System.out.println("Main finished -- JVM exits NOW, daemon is killed mid-loop");
    }
}
```
**Output (daemon is cut off mid-work, never reaches a clean stop):**
```
Main thread doing work...
Daemon running: 0
Daemon running: 1
Daemon running: 2
Main finished -- JVM exits NOW, daemon is killed mid-loop
```
Notice the daemon thread never prints "Daemon running: 3" or beyond — the instant `main()` returns, the JVM tears down immediately, mid-loop, with no exception, no `finally`, no warning.

> ⚠️ **Pitfall:** never put logic requiring guaranteed completion (writing a file, committing a transaction) on a daemon thread — the JVM can terminate it mid-operation with zero warning the instant the last user thread finishes. Also: `setDaemon(true)` must be called **before** `start()` — calling it after throws `IllegalThreadStateException`. A thread spawned by a daemon thread inherits daemon status by default, same for user threads.

## synchronized Reentrancy

If a thread already holds a lock and calls another method requiring the *same* lock, Java lets it right through instead of deadlocking the thread against itself.

```java
class Reentrant {
    synchronized void outer() {
        System.out.println("In outer");
        inner();
    }
    synchronized void inner() {
        System.out.println("In inner — same thread re-acquired the lock");
    }
}
new Reentrant().outer();
```
**Output:**
```
In outer
In inner — same thread re-acquired the lock
```
> Without reentrancy, this exact pattern would deadlock a thread against itself the moment `outer()` called `inner()`.

---

## Interview Q&A

**Q: Threads T1, T2, T3 — ensure T2 runs after T1 and T3 runs after T2.**
```java
Thread t1 = new Thread(task1);
Thread t2 = new Thread(() -> { try { t1.join(); } catch (InterruptedException e) {} task2.run(); });
Thread t3 = new Thread(() -> { try { t2.join(); } catch (InterruptedException e) {} task3.run(); });
t1.start(); t2.start(); t3.start();
```
`Thread.join()` blocks the calling thread until the target thread finishes — chaining `join()` calls enforces sequential ordering. (Modern alternative: `CompletableFuture.supplyAsync(task1).thenRun(task2).thenRun(task3)` — see the CompletableFuture guide.)
> ⚠️ **Pitfall:** all three threads can still be *started* in any order — it's the `join()` calls inside each thread's logic that enforce execution order. Don't conflate "start order" with "execution order."

**Q: Can we start a thread twice in Java?**
No — calling `start()` on a thread that's already been started (finished or not) throws `IllegalThreadStateException`. A `Thread` object represents a one-time execution lifecycle; you must create a new instance to run the logic again.
> ⚠️ **Pitfall:** this is exactly why thread pools (`ExecutorService`, Part 2) exist — they reuse worker *threads* to run many different *tasks* over time, rather than creating and discarding a `Thread` object per task.

**Q: Can we run a thread twice in Java?**
Calling `run()` directly (not `start()`) a second time is legal — but it just executes `run()`'s logic synchronously on the **current** thread, like any ordinary method call. It does not spawn a new thread. This is a classic beginner mistake — code compiles and "works" but runs single-threaded.

**Q: Process-based vs thread-based multitasking — what's the actual distinction?**
Covered above under "Program vs Process vs Thread." If asked to justify a design choice ("why not just use separate processes instead of threads"), lead with the isolation-vs-communication-cost tradeoff, not just "processes are heavier" — that's the actual engineering decision being probed.

**Q: Four common threading myths — what's actually true in each case?**
- *"`volatile` makes code thread-safe"* — false. Only guarantees visibility, not atomicity (see Part 5).
- *"`sleep()` releases the lock"* — false. A thread sleeping while holding a monitor keeps holding it the entire duration; only `wait()` releases it.
- *"`start()` immediately begins execution"* — false. `start()` only **requests** the OS scheduler run the new thread — no guarantee about *when*. Code right after `start()` in the calling thread can easily run before the new thread gets its first time slice.
- *"More threads always improve performance"* — false. Beyond a point, more threads increase context-switching, memory, and scheduling overhead, actively *reducing* throughput — the same tradeoff behind why `ThreadPoolExecutor` bounds pool size (Part 2) instead of spawning unboundedly.
> ⚠️ **Pitfall:** these four are worth having ready as a rapid-fire "myth or fact" list — interviewers often open with one as a quick calibration question before going deeper.

**Q: Why is creating a new Thread expensive, concretely?**
Three real costs: **(1) Native OS thread allocation** — a genuine system call. **(2) Stack allocation** — each thread gets its own dedicated stack (commonly ~512KB–1MB), reserved up front. **(3) Scheduler registration** — the thread must register with the OS scheduler before it's even eligible to run.
> ⚠️ **Pitfall:** "thread pools reuse threads" is the standard answer to why pools exist — but naming these three specific costs being amortized away is what separates a memorized rule from an understood one.

**Q: What happens when you call interrupt() on a thread that's blocked in sleep()/wait()? What if it's just running normal code?**
Covered above under "Interrupting Threads." Blocked in `sleep()`/`wait()`/`join()`: woken immediately, flag cleared, `InterruptedException` thrown. Running normal code: only the flag is set — the thread must periodically call `isInterrupted()` itself and choose to stop; nothing forces it to.

**Q: Thread.interrupted() vs someThread.isInterrupted() — what's the actual difference?**
Covered above. `Thread.interrupted()` is `static`, always targets the *current* thread, and **clears** the flag as a side effect of reading it. `isInterrupted()` is an instance method, can check *any* thread, and never clears anything. Calling both in the same statement, the order changes what the second call sees.
