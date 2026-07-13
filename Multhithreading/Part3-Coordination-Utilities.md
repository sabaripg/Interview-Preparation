# Part 3 — Coordination Utilities

> CountDownLatch, CyclicBarrier, Phaser, Semaphore, Exchanger, Condition — tools for making threads wait on or signal each other. Interview Q&A at the end.

## CountDownLatch

**What it does:** lets one or more threads wait until a set number of "events" have all happened, before proceeding. Think of it as a countdown timer starting at N — everyone waiting is released the instant it hits zero.

**Key features:** initialized with a fixed count; each event decrements it via `countDown()`; waiting threads block on `await()` until zero; **one-time use** — cannot be reset.

```java
CountDownLatch latch = new CountDownLatch(3);
for (int i = 0; i < 3; i++) {
    Thread t = new Thread(new WorkerThread(latch));
    t.start();
}
System.out.println("Main waiting...");
latch.await();
System.out.println("All operations done, resuming.");

class WorkerThread implements Runnable {
    CountDownLatch latch;
    WorkerThread(CountDownLatch latch) { this.latch = latch; }
    public void run() {
        try {
            Thread.sleep(1000);
            System.out.println(Thread.currentThread().getName() + " finished");
        } catch (InterruptedException e) {
        } finally {
            latch.countDown();
        }
    }
}
```
**One possible output:**
```
Main waiting...
Thread-0 finished
Thread-1 finished
Thread-2 finished
All operations done, resuming.
```

**Waiting with a timeout** — instead of waiting forever, give up after a bound:
```java
if (latch.await(5, TimeUnit.SECONDS)) {
    System.out.println("All workers finished within timeout.");
} else {
    System.out.println("Timeout reached before all workers finished.");
}
```

## CyclicBarrier

**What it does:** makes a fixed group of threads all wait at a common checkpoint until every one has arrived, then releases them all at once — and unlike `CountDownLatch`, it's **reusable**: once everyone passes through, it resets for the next round.

```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("All parties arrived!"));
Runnable task = () -> {
    try {
        System.out.println(Thread.currentThread().getName() + " waiting at barrier");
        barrier.await();
    } catch (Exception e) {}
};
for (int i = 0; i < 3; i++) new Thread(task).start();
```
**One possible output:**
```
Thread-0 waiting at barrier
Thread-1 waiting at barrier
Thread-2 waiting at barrier
All parties arrived!
```

## CountDownLatch vs CyclicBarrier — the worked comparison

| | CountDownLatch | CyclicBarrier |
|---|---|---|
| Who's waiting? | Threads wait for OTHER threads (two roles: waiter + counter) | Same threads wait on EACH OTHER (one role: everyone both works and waits) |
| After it fires once | Dead permanently — need a new object for a repeat | Auto-resets — same object works for round 2, round 3, forever |
| Real-world shape | "Main waits for 3 workers to finish setup, once" | "3 workers keep pace with each other across many repeated rounds" |

> ⚠️ **Pitfall:** the reusability distinction is the core one, and getting it backward is a very common mistake. Clean mental image: a **CountDownLatch is a finish line torn down after one race**; a **CyclicBarrier is a finish line that automatically rebuilds itself for the next race, with the same runners.**

## Phaser

**What it does:** a more flexible version of `CountDownLatch`/`CyclicBarrier` for multi-phase work where the number of participating threads can change between phases (dynamic registration/deregistration — the other two require a fixed party count).

```java
Phaser phaser = new Phaser(1); // register self
System.out.println("Phase before: " + phaser.getPhase());
phaser.arriveAndAwaitAdvance(); // signal arrival, wait for everyone else registered
System.out.println("Phase after: " + phaser.getPhase());
phaser.arriveAndDeregister(); // signal arrival and leave, so future phases don't wait on this thread
```
**Output:**
```
Phase before: 0
Phase after: 1
```
> ⚠️ **Pitfall:** `Phaser` is genuinely more powerful but rarely needed in typical application code — `CountDownLatch`/`CyclicBarrier` cover the vast majority of real coordination needs. Reach for `Phaser` specifically when the dynamic-party-count requirement is real, not as a default upgrade.

## Semaphore

**What it does:** controls how many threads can be inside a critical section **at the same time** — not just 1 like a lock, but any number N you choose. Think of N numbered entry passes: a thread must hold one to enter, gives it back when done; if all passes are taken, the next thread waits.

```java
Semaphore semaphore = new Semaphore(2); // 2 permits
Runnable task = () -> {
    try {
        semaphore.acquire();
        System.out.println(Thread.currentThread().getName() + " accessing resource");
        Thread.sleep(500);
    } catch (InterruptedException e) {
    } finally {
        semaphore.release();
        System.out.println(Thread.currentThread().getName() + " released resource");
    }
};
for (int i = 0; i < 3; i++) new Thread(task).start();
```
**One possible output (3 threads compete for 2 permits — 3rd waits):**
```
Thread-0 accessing resource
Thread-1 accessing resource
Thread-0 released resource
Thread-2 accessing resource
Thread-1 released resource
Thread-2 released resource
```

### Semaphore vs Mutex

| | Mutex | Semaphore |
|---|---|---|
| **Permits** | Binary — locked or unlocked (1 permit) | Counting — N permits |
| **Ownership** | Owned by the thread that locked it — only that thread can unlock it | No ownership — any thread can call `release()`, even one that never called `acquire()` |
| **Purpose** | Mutual exclusion | Resource-count limiting |
| **Java equivalent** | `synchronized`, `ReentrantLock` | `java.util.concurrent.Semaphore` |
| **Reentrant?** | Yes for `ReentrantLock`/`synchronized` | Not inherently — re-acquiring just consumes another permit |

> ⚠️ **Pitfall — the point interviewers actually probe:** a Mutex has an **owner**; a Semaphore does not. This is why a Semaphore can be used for signaling *between* threads (thread A acquires, thread B releases) — a Mutex cannot, since only the locking thread is allowed to unlock it.

```java
public static void main(String[] args) throws InterruptedException {
    Semaphore signal = new Semaphore(0); // starts with 0 permits — waiter blocks immediately

    Thread waiter = new Thread(() -> {
        try {
            System.out.println("Waiter: waiting for signal...");
            signal.acquire();
            System.out.println("Waiter: signal received, proceeding");
        } catch (InterruptedException e) {}
    });
    Thread signaler = new Thread(() -> {
        try { Thread.sleep(1000); } catch (InterruptedException e) {}
        System.out.println("Signaler: sending signal");
        signal.release(); // signaler never called acquire()
    });
    waiter.start();
    signaler.start();
}
```
**Output:**
```
Waiter: waiting for signal...
Signaler: sending signal
Waiter: signal received, proceeding
```

## Exchanger

**What it does:** a synchronization point for exactly **two** threads to swap objects — each calls `exchange(myObject)`, blocking until the other arrives, then both receive the other's object and proceed.

```java
Exchanger<String> exchanger = new Exchanger<>();
Thread t1 = new Thread(() -> {
    try {
        String received = exchanger.exchange("Data from T1");
        System.out.println("T1 received: " + received);
    } catch (InterruptedException e) {}
});
Thread t2 = new Thread(() -> {
    try {
        String received = exchanger.exchange("Data from T2");
        System.out.println("T2 received: " + received);
    } catch (InterruptedException e) {}
});
t1.start(); t2.start();
```
**Output:**
```
T1 received: Data from T2
T2 received: Data from T1
```
> ⚠️ **Pitfall:** be honest that this is rarely reached for in real application code — `BlockingQueue` covers the vast majority of producer/consumer needs even for two parties. Mention `Exchanger` to show breadth, don't overstate how often it comes up in practice.

## Condition

**What it does:** the modern, explicit-lock version of `wait()`/`notify()`. A `Condition` represents one specific "thing threads might be waiting for," created via `lock.newCondition()`. `condition.await()` releases the lock and waits; `condition.signal()`/`signalAll()` wakes waiters.

```java
Lock lock = new ReentrantLock();
Condition notEmpty = lock.newCondition();
Condition notFull = lock.newCondition();

// producer
lock.lock();
try {
    while (queue.isFull()) notFull.await();
    queue.add(item);
    notEmpty.signal();
} finally { lock.unlock(); }
```

**The key improvement over `wait()`/`notifyAll()`:** a single `Lock` can create **multiple** `Condition` objects, each representing a different wait condition. A producer/consumer buffer can have a separate `notFull` condition (producers wait here when full) and `notEmpty` condition (consumers wait here when empty), rather than one shared monitor where `notifyAll()` wakes *every* waiter regardless of which condition they were actually waiting on.

> ⚠️ **Pitfall:** same "always loop, never `if`" rule applies — `while (condition) await();`, not `if`, since a signaled thread must re-verify the condition after re-acquiring the lock. The multiple-conditions capability is the actual "why use this over wait/notify" answer — name the specific precision gain, don't just say "it's newer."

## Missed Signals — a Condition Pitfall

**What it is:** a signal is only useful if *someone is already waiting* when it fires. `signal()`/`notify()` don't queue up or get remembered for later — if the signaling thread calls it before any thread has called `await()`/`wait()`, the signal simply vanishes into nothing, and the eventual waiter blocks forever with no signal left to receive.

```java
class MissedSignalExample {
    static void example() throws InterruptedException {
        ReentrantLock lock = new ReentrantLock();
        Condition condition = lock.newCondition();

        Thread signaller = new Thread(() -> {
            lock.lock();
            try {
                condition.signal();
                System.out.println("Sent signal");
            } finally { lock.unlock(); }
        });

        Thread waiter = new Thread(() -> {
            lock.lock();
            try {
                condition.await();       // <-- nothing to catch, the signal already fired and vanished
                System.out.println("Received signal");
            } catch (InterruptedException ie) {
                // handle interruption
            } finally { lock.unlock(); }
        });

        signaller.start();
        signaller.join();   // signaller finishes and exits BEFORE waiter even starts
        waiter.start();
        waiter.join();
        System.out.println("Program exiting.");
    }
}
```
**What actually happens:** `signaller` runs to completion — signals, then exits — entirely before `waiter` even starts. By the time `waiter` calls `await()`, there is no pending signal to receive; nothing will ever wake it. `waiter.join()` blocks forever and the program never reaches "Program exiting."

**Why this happens — the root cause:** `Condition`/`wait`-`notify` signals are not state. There's no counter, no memory of "a signal happened at 10:02am" — a `signal()` call only has any effect on threads that are *already* parked in `await()` at that exact moment. If nobody's listening, the signal is simply thrown away.

**Fix 1 — the idiomatic fix: encode the "did it happen" state in a boolean, and loop on it** (the same `while (condition) await()` rule from the section above):
```java
boolean signalled = false; // shared, guarded by the same lock

// signaller
lock.lock();
try {
    signalled = true;
    condition.signal();
} finally { lock.unlock(); }

// waiter
lock.lock();
try {
    while (!signalled) {   // if the signal already happened, this loop is skipped entirely — no missed signal possible
        condition.await();
    }
} finally { lock.unlock(); }
```
This is the real reason `await()`/`wait()` must always be called in a `while` loop checking real, persisted state — not just so a spuriously-woken thread re-checks, but because it's the only thing that protects you from a signal that arrived *before* you started waiting at all.

**Fix 2 — use a `Semaphore` instead, when the goal is pure signaling:** unlike a `Condition`, a `Semaphore`'s permits **are** state — `release()` before anyone `acquire()`s just leaves a permit sitting there, available whenever a thread eventually asks for it. Nothing is lost.
```java
Semaphore semaphore = new Semaphore(0); // starts empty — a waiter blocks until a permit exists

Thread signaller = new Thread(() -> {
    semaphore.release(); // leaves a permit available, even with zero waiters right now
    System.out.println("Sent signal");
});
Thread waiter = new Thread(() -> {
    try {
        semaphore.acquire(); // picks up the permit whenever it runs — even if that's well after release()
        System.out.println("Received signal");
    } catch (InterruptedException ie) { }
});
```
See the "Semaphore vs Mutex" section above for the full runnable version of this exact pattern — it's the same underlying fix.

> ⚠️ **Pitfall:** this is precisely the mistake that separates "I know the `Condition` API" from "I understand what it actually guarantees." `Condition`/`wait`/`notify` coordinate around a **moment in time**; a `Semaphore`'s permit count coordinates around **durable state**. Whenever the order between "signal" and "start waiting" isn't guaranteed by your code's structure, prefer a mechanism with real state (a boolean + `while` loop, or a `Semaphore`/`BlockingQueue`) over a raw signal.

---

## Interview Q&A

**Q: Difference between CountDownLatch and CyclicBarrier — when would you use each?**
Covered above with worked examples — see the comparison table.

**Q: What is the Phaser class, and how does it differ from CountDownLatch/CyclicBarrier?**
Covered above.

**Q: What is a Semaphore, and how does it differ from a lock?**
A `Semaphore` maintains a set number of permits — `acquire()` blocks if none available, `release()` returns one. Unlike a lock (exactly one thread at a time), a semaphore can allow **N** concurrent threads.
> ⚠️ **Pitfall:** a `Semaphore` with permit count 1 behaves like a mutex, a useful mental bridge — but the key generalization is permits > 1, letting multiple threads through simultaneously, which no plain lock can express.

**Q: What is an Exchanger, and when would you actually use one?**
Covered above.

**Q: What is the Condition interface, and how does it improve on wait()/notify()?**
Covered above.

**Q: What is a "missed signal," and how do you avoid it?**
Covered above under "Missed Signals." A `signal()`/`notify()` call has no effect on threads that aren't already waiting at that exact moment — signals aren't remembered or queued. Fix: never rely on signal timing alone; guard `await()`/`wait()` with a real, persisted boolean condition checked in a `while` loop, or use a `Semaphore`/`BlockingQueue` whose state (permits, queued items) survives regardless of when threads show up.

**Q: Write the Producer/Consumer problem using wait and notify.**
```java
class BoundedBuffer<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    BoundedBuffer(int capacity) { this.capacity = capacity; }

    synchronized void produce(T item) throws InterruptedException {
        while (queue.size() == capacity) {
            wait(); // buffer full, wait for consumer
        }
        queue.add(item);
        notifyAll(); // wake up any waiting consumers
    }

    synchronized T consume() throws InterruptedException {
        while (queue.isEmpty()) {
            wait(); // buffer empty, wait for producer
        }
        T item = queue.poll();
        notifyAll(); // wake up any waiting producers
        return item;
    }
}
```
Both `wait()` calls are inside a `while` loop, not `if` — essential, since a thread can wake spuriously or find the condition still false by the time it re-acquires the lock. `notifyAll()` (not `notify()`) is the safer default with multiple producers/consumers.
> ⚠️ **Pitfall:** the `while` (not `if`) around `wait()` is the single most important correctness detail here. In real production code, `BlockingQueue` (e.g. `ArrayBlockingQueue`, see Part 6) already implements this correctly and should be preferred over hand-rolling it.

**Q: wait() vs sleep() — what's the actual difference?**
`wait()` releases the object's monitor lock while waiting, must be called inside a `synchronized` block (else `IllegalMonitorStateException`), and needs an explicit `notify()`/`notifyAll()` or timeout to resume. `sleep()` does **not** release any locks held, doesn't require `synchronized`, and resumes automatically after its duration.
> ⚠️ **Pitfall:** the lock-release difference has real consequences — calling `sleep()` while holding a lock keeps it held the entire duration, potentially blocking every other thread waiting on it. This is exactly why `wait()`/`notify()` is correct for producer/consumer coordination rather than `sleep()` in a polling loop.
