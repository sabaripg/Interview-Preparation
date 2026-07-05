# Interview-Preparation

# Multithreading — Part 1

**Topics covered:**
- Process and Thread
- Ways to Create a Thread
- Thread Priority
- Monitor Lock
- wait() vs notify() vs notifyAll()
- Producer-Consumer using wait/notify
- Daemon Thread
- ❌ Why stop(), suspend(), and resume() were deprecated
- Thread Pool
  - ThreadPoolExecutor
  - ForkJoinPool
  - ScheduledThreadPoolExecutor
- BlockingQueue
- ThreadFactory
- RejectedExecutionHandler

> Note: This part covers Process/Thread fundamentals through Daemon Threads. Thread Pool, BlockingQueue, ThreadFactory, and RejectedExecutionHandler are covered in **Part 2**.

---

## Process and Thread

Before understanding multithreading, let's first understand **Process** and **Thread**.

### Process

In Java, a **process** is an executing instance of a Java application.

When you run a `.java` program using the JVM, the OS creates a process and allocates resources such as memory (heap, stack, method area), CPU time, and file handles. Each process runs in its own memory space and is independent of other processes.

**Example:** If you run two Java programs (`ProgramA.java` and `ProgramB.java`), the OS creates two separate processes. Each process gets its own JVM instance and memory allocation.

**How much memory does a process get?**

When you run a Java program (`java MainClassName`), the OS creates a new process and starts a new JVM instance. This JVM process gets memory divided into different areas:

- **Heap Memory** — for objects
- **Stack Memory** — for method calls, local variables, references
- **Method Area / Metaspace** — for class metadata, static variables
- **Native Memory** — for JVM internals, threads, direct buffers

### Thread

A **thread** is known as a lightweight process — the smallest sequence of instructions that the CPU can execute independently. One process can have multiple threads.

When a process is created, it starts with one thread, known as the **main thread**. From there, we can create additional threads to perform tasks concurrently.

```java
public class MultithreadingLearning {
    public static void main(String[] args) {
        System.out.println("Thread Name: " + Thread.currentThread().getName());
    }
}
```
**Output:**
```
Thread Name: main
```

### Definition of Multithreading

Multithreading allows a program to perform multiple tasks at the same time. Multiple threads share the same resources (like memory space) but can still execute independently.

**Benefits:**
- Improved performance through task parallelism
- Better responsiveness
- Resource sharing

**Challenges:**
- Concurrency issues like deadlock and data inconsistency
- Synchronization overhead
- Harder to test and debug

### Context Switching

Context switching in Java refers to the process where the CPU pauses execution of one thread — saving its state — and resumes another thread by restoring its saved state.

Although we say "in Java," the actual context switching is performed by the **OS scheduler**, since Java threads map to native OS threads.

---

## Thread Creation Ways

There are two ways to create a thread in Java:
1. Implementing the `Runnable` interface
2. Extending the `Thread` class

### Way 1 — Implementing Runnable

**Step 1: Create a Runnable object**

```java
public class MultithreadingLearning implements Runnable {
    @Override
    public void run() {
        System.out.println("Code executed by thread: " + Thread.currentThread().getName());
    }
}
```

**Step 2: Pass it to a Thread and start it**

```java
public class Main {
    public static void main(String[] args) {
        System.out.println("Going inside main method: " + Thread.currentThread().getName());
        MultithreadingLearning runnableObj = new MultithreadingLearning();
        Thread thread = new Thread(runnableObj);
        thread.start();
        System.out.println("Finish main method: " + Thread.currentThread().getName());
    }
}
```

**Output:**
```
Going inside main method: main
Finish main method: main
Code executed by thread: Thread-0
```

> Note: Because `thread.start()` is asynchronous, the exact interleaving of "Finish main method" and "Code executed by thread" isn't guaranteed — the scheduler decides. The output above is one possible ordering, not a guarantee.

### Way 2 — Extending Thread

**Step 1: Create a Thread subclass**

```java
public class MultithreadingLearning extends Thread {
    @Override
    public void run() {
        System.out.println("Code executed by thread: " + Thread.currentThread().getName());
    }
}
```

**Step 2: Instantiate and start**

```java
public class Main {
    public static void main(String[] args) {
        System.out.println("Going inside main method: " + Thread.currentThread().getName());
        MultithreadingLearning myThread = new MultithreadingLearning();
        myThread.start();
        System.out.println("Finish main method: " + Thread.currentThread().getName());
    }
}
```

**Output:**
```
Going inside main method: main
Finish main method: main
Code executed by thread: Thread-0
```

### Why Two Ways to Create Threads?

- A class can implement **multiple** interfaces, but
- A class can extend only **one** class

So Java provides:
- `implements Runnable` — **preferred**, since it doesn't use up your one shot at inheritance and separates the task from the thread mechanism
- `extends Thread` — less flexible, since your class can no longer extend anything else

### Important Notes (Interview Gold ⭐)

- `stop()` is **deprecated** ❌ (unsafe — can leave shared objects in an inconsistent state; kills the thread mid-operation without releasing locks properly)
- `start()` → creates a **new call stack** (a new OS-level thread) → internally invokes `run()` on that new thread
- Calling `run()` directly ❌ → executes on the **current** thread like a normal method call — no new thread is created
- Thread state transitions are managed by the **Thread Scheduler + JVM**, not by application code directly

---

## Thread Lifecycle States

| State | Description |
|---|---|
| **New** | Thread has been created but not started. It's just an object in memory. |
| **Runnable** | Thread is ready to run and waiting for CPU time. |
| **Running** | Thread is actively executing its code. |
| **Blocked** | Thread is waiting to acquire a lock held by another thread (e.g., trying to enter a synchronized block), or waiting on blocking I/O. It releases all monitor locks it currently holds while blocked. |
| **Waiting** | Thread enters this state after calling `wait()` (with no timeout), `join()` (no timeout), or `LockSupport.park()`. It returns to Runnable once `notify()`/`notifyAll()` is called or the joined thread finishes. It releases the monitor lock while waiting. |
| **Timed Waiting** | Thread waits for a bounded period via calls like `sleep(ms)`, `wait(ms)`, or `join(ms)`. **Important distinction:** `sleep(ms)` does **not** release any monitor lock it holds, but `wait(ms)` **does** release the lock while waiting — both states are labeled "Timed Waiting," but their locking behavior differs. |
| **Terminated** | The thread has finished executing (or was stopped) and cannot be started again. |

---

## Monitor Lock

Before the wait/notify examples, let's understand the **monitor lock**.

A monitor lock ensures that only one thread at a time can execute a particular section of code (a `synchronized` block or method) on the same object.

```java
public class MonitorLockExample {

    public synchronized void task1() {
        try {
            System.out.println("Inside task1");
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            // handle exception
        }
    }

    public void task2() {
        System.out.println("task2, but before synchronized");
        synchronized (this) {
            System.out.println("task2, inside synchronized");
        }
    }

    public void task3() {
        System.out.println("task3");
    }

    public static void main(String[] args) {
        MonitorLockExample obj = new MonitorLockExample();

        Thread t1 = new Thread(obj::task1);
        Thread t2 = new Thread(obj::task2);
        Thread t3 = new Thread(obj::task3);

        t1.start();
        t2.start();
        t3.start();
    }
}
```

**One possible output:**
```
Inside task1
task3
task2, but before synchronized
task2, inside synchronized
```

> Note: `task3()` isn't synchronized at all, so it can print at any point regardless of the lock held by `task1()`. `task2()`'s first `println` (before entering the `synchronized` block) can also run immediately — only the line *inside* the `synchronized (this)` block has to wait for `task1()` to release the lock. The exact interleaving is not guaranteed by the JVM; only the *mutual exclusion* on the synchronized sections is guaranteed.

### What Happens Here?

Both threads calling a `synchronized` method on the *same object* must take turns holding that object's monitor lock — one thread completes (or otherwise exits the synchronized code) before the next can enter.

```java
class SharedResource {
    public synchronized void printMessage(String msg) {
        System.out.println("[" + msg);
        try {
            Thread.sleep(1000); // simulate work
        } catch (InterruptedException e) {}
        System.out.println("]");
    }
}

public class Main {
    public static void main(String[] args) {
        SharedResource resource = new SharedResource();
        Thread t1 = new Thread(() -> resource.printMessage("Hello"));
        Thread t2 = new Thread(() -> resource.printMessage("World"));
        t1.start();
        t2.start();
    }
}
```

Since `printMessage()` is synchronized, only one thread at a time can hold the monitor lock on `resource`. The second thread must wait until the first releases the lock.

**Another example (same idea, longer sleep to make the interleaving obvious):**

```java
public class MonitorLockExample {

    public synchronized void task1() {
        try {
            System.out.println("inside task1");
            Thread.sleep(10000);
            System.out.println("task1 completed");
        } catch (InterruptedException e) {
            // handle exception
        }
    }

    public void task2() {
        System.out.println("task2, but before synchronized");
        synchronized (this) {
            System.out.println("task2, inside synchronized");
        }
    }

    public void task3() {
        System.out.println("task3");
    }

    public static void main(String[] args) {
        MonitorLockExample obj = new MonitorLockExample();
        Thread t1 = new Thread(obj::task1);
        Thread t2 = new Thread(obj::task2);
        Thread t3 = new Thread(obj::task3);
        t1.start();
        t2.start();
        t3.start();
    }
}
```

**One possible output:**
```
inside task1
task3
task2, but before synchronized
task1 completed
task2, inside synchronized
```

---

## Producer-Consumer Using wait() / notifyAll()

> Note: This is a **simplified, single-item** version of the classic producer-consumer problem (a boolean flag instead of a real bounded queue/buffer) — useful for understanding `wait()`/`notifyAll()` mechanics. A production version would use an actual bounded buffer (e.g., `BlockingQueue`, covered in Part 2) to hold multiple items safely.

```java
class SharedResource {

    private boolean isItemPresent = false;

    public synchronized void addItem() {
        isItemPresent = true;
        System.out.println("Producer produced item");
        notifyAll(); // inform consumer(s) that an item is available
    }

    public synchronized void consumeItem() {
        System.out.println("Consumer checking for item");

        while (!isItemPresent) {   // guard against spurious wakeups
            try {
                wait();  // releases lock, enters WAITING state
            } catch (InterruptedException e) {
                // handle exception
            }
        }

        System.out.println("Consumer consumed item");
        isItemPresent = false;
    }
}
```

**Producer:**
```java
class ProducerTask implements Runnable {

    SharedResource sharedResource;

    ProducerTask(SharedResource resource) {
        this.sharedResource = resource;
    }

    @Override
    public void run() {
        try {
            Thread.sleep(2000);  // simulate delay
        } catch (InterruptedException e) {
            // handle exception
        }
        sharedResource.addItem();
    }
}
```

**Consumer:**
```java
class ConsumerTask implements Runnable {

    SharedResource sharedResource;

    ConsumerTask(SharedResource resource) {
        this.sharedResource = resource;
    }

    @Override
    public void run() {
        System.out.println("Consumer Thread: " + Thread.currentThread().getName());
        sharedResource.consumeItem();
    }
}
```

**Main:**
```java
public class Main {
    public static void main(String[] args) {
        System.out.println("Main method starts");

        SharedResource sharedResource = new SharedResource();

        Thread producerThread = new Thread(new ProducerTask(sharedResource));
        Thread consumerThread = new Thread(new ConsumerTask(sharedResource));

        producerThread.start();
        consumerThread.start();

        System.out.println("Main method ends");
    }
}
```

**One possible output:**
```
Main method starts
Main method ends
Consumer Thread: Thread-1
Consumer checking for item
Producer produced item
Consumer consumed item
```

> ⚠️ Why the `while` loop instead of `if`? Two reasons: (1) **spurious wakeups** — the JVM spec allows a waiting thread to wake up without an actual `notify()` call, so the condition must be re-checked; (2) with `notifyAll()` and multiple consumers, more than one thread can wake up but only one should actually proceed — re-checking the condition after waking prevents the others from consuming a non-existent item.

---

## wait() vs notify() vs notifyAll()

These methods live on `Object` (not `Thread`) and are used for inter-thread communication. They can only be called from within a `synchronized` block/method on the object being waited on — calling them outside a synchronized context throws `IllegalMonitorStateException`.

### wait()

- Makes the current thread wait until another thread calls `notify()`/`notifyAll()` on the same object.
- Releases the monitor lock and moves the thread to the `WAITING` (or `TIMED_WAITING` if a timeout is given) state.

```java
synchronized (obj) {
    obj.wait();
}
```

Flow: Thread acquires lock → calls `wait()` → releases lock → enters waiting state.

### notify()

- Wakes **one** arbitrary thread waiting on the object's monitor (the JVM does not guarantee any particular selection order).
- The woken thread cannot proceed until it **re-acquires** the lock.

```java
synchronized (obj) {
    obj.notify();
}
```

### notifyAll()

- Wakes **all** threads waiting on the object's monitor.
- All woken threads move to Runnable, but only one at a time can actually re-acquire the lock and proceed.

```java
synchronized (obj) {
    obj.notifyAll();
}
```

> ⚠️ **Prefer `notifyAll()` over `notify()`** in most real code — `notify()` can wake the "wrong" thread when multiple threads are waiting for different conditions on the same object, causing missed signals/deadlock-like hangs. `notifyAll()` is safer by default, at the cost of some extra wakeups.

---

## Thread Priority

Thread priority is a **hint** to the thread scheduler about the relative importance of a thread. A higher-priority thread is *more likely* to get CPU time before a lower-priority one — but this is **not guaranteed**, and actual scheduling behavior is OS-dependent.

**Priority range:** 1 to 10

| Constant | Value | Meaning |
|---|---|---|
| `Thread.MIN_PRIORITY` | 1 | Lowest priority |
| `Thread.NORM_PRIORITY` | 5 | Default priority |
| `Thread.MAX_PRIORITY` | 10 | Highest priority |

```java
class PriorityExample {
    public static void main(String[] args) {

        Thread t1 = new Thread(() -> System.out.println("Thread 1 running"));
        Thread t2 = new Thread(() -> System.out.println("Thread 2 running"));

        t1.setPriority(Thread.MIN_PRIORITY); // 1
        t2.setPriority(Thread.MAX_PRIORITY); // 10

        t1.start();
        t2.start();
    }
}
```

> ⚠️ Don't design real concurrency logic around thread priority — it's a scheduling *hint*, not a scheduling *guarantee*, and behaves inconsistently across JVMs/OSes.

---

## Daemon Thread

A **daemon thread** is a background/service thread that supports user (non-daemon) threads. The JVM does **not** wait for daemon threads to finish before exiting — once all non-daemon (user) threads complete, the JVM shuts down even if daemon threads are still running.

**Common daemon threads in the JVM:**
- Garbage Collector
- Finalizer thread
- Signal dispatcher

```java
class DaemonExample {
    public static void main(String[] args) {

        Thread daemonThread = new Thread(() -> {
            while (true) {
                System.out.println("Daemon thread running...");
            }
        });

        daemonThread.setDaemon(true); // mark as daemon
        daemonThread.start();

        System.out.println("Main thread finished");
    }
}
```

### Important Rules

1️⃣ **`setDaemon()` must be called before `start()`**
```java
thread.start();
thread.setDaemon(true); // ❌ throws IllegalThreadStateException
```

2️⃣ **The JVM doesn't wait for daemon threads** — it exits as soon as all non-daemon threads finish, potentially killing daemon threads mid-task.

3️⃣ **Daemon threads are meant for background support tasks**, such as:
- Garbage collection
- Monitoring
- Logging
- Cleanup tasks

---

*Corrections made from the original draft: fixed "Demon Thread" → "Daemon Thread"; removed a duplicated paragraph in the Daemon Thread section; corrected the Timed Waiting lock-release behavior (sleep() vs wait(ms) differ); clarified that the producer-consumer example is a simplified single-item version, not a true bounded buffer; fixed a missing `while`-loop rationale (spurious wakeups); added missing `catch` types (`InterruptedException` instead of generic `Exception`) for accuracy; noted that thread-interleaving outputs shown are one possible ordering, not guaranteed.*
