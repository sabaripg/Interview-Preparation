# Part 7 — Fork/Join & Parallelism

> Concurrency vs parallelism, the ForkJoinPool framework, RecursiveTask, work-stealing. Interview Q&A at the end.

## Concurrency vs Parallelism

**Concurrency** — structuring a program to *deal with* multiple tasks in progress at overlapping time periods, potentially interleaved on a single core (time-slicing) rather than literally simultaneous. It's about managing multiple things happening logically at once.

**Parallelism** — actually *executing* multiple tasks at the exact same instant, requiring multiple cores. It's one specific way of achieving concurrency's goals, but not the only way.

A single-core machine can be concurrent (via context-switching) but never truly parallel; a multi-core machine can be both.

> **The cleanest framing (Rob Pike):** "Concurrency is about *dealing with* lots of things at once; parallelism is about *doing* lots of things at once." Concurrency is a program-structure concept; parallelism is an execution/hardware concept.

## Fork/Join Framework vs Traditional Thread Pools

**What it does:** `ForkJoinPool` is designed specifically for **divide-and-conquer** workloads — a task recursively splits itself into smaller subtasks (`fork()`), and results are combined (`join()`) — well-suited for recursive algorithms and parallel stream operations.

**Work-stealing:** each worker thread has its own local deque of tasks. If a thread finishes its own queue early, it "steals" tasks from the *tail* of another busy thread's queue, keeping all CPU cores well-utilized without a single central task queue becoming a contention bottleneck.

Traditional thread pools (`ThreadPoolExecutor`, Part 2) use a single shared task queue — simpler, well-suited for independent, non-recursive tasks, but not optimized for tasks that spawn further subtasks the way Fork/Join is.

`Stream.parallelStream()` uses the common `ForkJoinPool` under the hood by default.

> ⚠️ **Pitfall:** work-stealing is the specific mechanism that makes Fork/Join efficient for recursive/uneven workloads — name it explicitly rather than just saying "it's for parallel tasks," since that's the actual differentiator from a plain thread pool.

## RecursiveTask — fork() and join() in Detail

```java
class FibonacciTask extends RecursiveTask<Integer> {
    private final int n;
    FibonacciTask(int n) { this.n = n; }
    protected Integer compute() {
        if (n <= 1) return n;
        FibonacciTask f1 = new FibonacciTask(n - 1);
        f1.fork(); // schedule asynchronously on the pool
        FibonacciTask f2 = new FibonacciTask(n - 2);
        return f2.compute() + f1.join(); // compute f2 on this thread, then wait for f1's result
    }
    public static void main(String[] args) {
        ForkJoinPool pool = new ForkJoinPool();
        System.out.println("Fibonacci(10) = " + pool.invoke(new FibonacciTask(10)));
    }
}
```
**Output:**
```
Fibonacci(10) = 55
```

`RecursiveTask<V>` (returns a result) and `RecursiveAction` (no result) are the two `ForkJoinTask` subclasses you extend to define divide-and-conquer work. `fork()` schedules a subtask asynchronously (potentially picked up by another worker via work-stealing); `join()` blocks until that subtask completes.

> ⚠️ **Pitfall:** computing one half **directly** on the current thread (not forking both halves) is the idiomatic pattern — forking every single subtask, including one you could just compute inline, adds unnecessary scheduling overhead for no parallelism benefit. Fork one half, compute the other half inline, then join the forked half.

---

## Interview Q&A

**Q: Difference between Concurrency and Parallelism?**
Covered above.

**Q: How is the Fork/Join framework different from traditional thread pools?**
Covered above.

**Q: Explain ForkJoinPool's RecursiveTask specifically — how does fork()/join() work?**
Covered above.
