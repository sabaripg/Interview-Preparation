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

## RecursiveAction — the no-result sibling of RecursiveTask

**What it is:** the other `ForkJoinTask` subclass — same fork/join divide-and-conquer model as `RecursiveTask`, but for work that doesn't produce a value (e.g. mutating an array in place, printing, side-effecting work).

```java
class IncrementAction extends RecursiveAction {
    private final int[] array;
    private final int start, end;
    static final int THRESHOLD = 2;

    IncrementAction(int[] array, int start, int end) {
        this.array = array; this.start = start; this.end = end;
    }

    protected void compute() {
        if (end - start <= THRESHOLD) {
            for (int i = start; i < end; i++) array[i]++;   // base case: do the work directly
            return;
        }
        int mid = (start + end) / 2;
        IncrementAction left = new IncrementAction(array, start, mid);
        IncrementAction right = new IncrementAction(array, mid, end);
        invokeAll(left, right);   // forks both, waits for both — convenience method for the fork()+fork()+join()+join() pattern
    }

    public static void main(String[] args) {
        int[] data = {1, 2, 3, 4, 5, 6};
        new ForkJoinPool().invoke(new IncrementAction(data, 0, data.length));
        System.out.println(Arrays.toString(data));
    }
}
```
**Output:**
```
[2, 3, 4, 5, 6, 7]
```
> ⚠️ **Pitfall — the one-line answer interviewers want:** `RecursiveTask<V>`'s `compute()` returns `V`; `RecursiveAction`'s `compute()` returns `void`. Same framework, same work-stealing, same fork/join mechanics — the only difference is whether the leaf computation produces a value you need back.

## Sizing the ForkJoinPool

`ForkJoinPool` defaults its parallelism to `Runtime.getRuntime().availableProcessors()` — one worker per CPU core, which makes sense for the **CPU-bound** divide-and-conquer work this framework targets. This is a meaningfully different sizing rule from the I/O-bound `ThreadPoolExecutor` sizing discussed in Part 2, and mixing the two mental models is a common mistake.

```java
ForkJoinPool customPool = new ForkJoinPool(4); // explicit parallelism level, instead of the default
```
> ⚠️ **Pitfall:** never run **blocking I/O** inside a `ForkJoinPool` task (including the common pool used by `parallelStream()` and `CompletableFuture`) — with only as many workers as CPU cores, a few blocked tasks can starve the pool for every other concurrent user of it in the JVM. That's precisely why `CompletableFuture`'s I/O-bound examples (see the dedicated guide) push you toward a separate custom `Executor` instead.

## Custom Thread Pool for Parallel Streams

**The problem:** `list.parallelStream()` always runs on the JVM-wide **common** `ForkJoinPool` — there's no `parallelStream(executor)` overload. Every parallel stream in your entire JVM (plus any `CompletableFuture` using the default async methods) shares that same pool. A long-running or blocking parallel stream operation can starve every other concurrent user of the common pool, and you can't simply hand it a differently-sized or isolated pool through the Streams API itself.

**The workaround:** submit the stream pipeline as a task to your own `ForkJoinPool` instance — since `ForkJoinTask.fork()`/`invoke()` execution picks up the *ambient* pool of whichever thread submitted the work, wrapping the whole pipeline in `customPool.submit(...)` makes the stream run on your pool instead of the common one.

```java
public class CustomPoolParallelStream {
    public static void main(String[] args) throws Exception {
        ForkJoinPool customPool = new ForkJoinPool(4); // isolated from the common pool

        List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8);

        int sum = customPool.submit(() ->
            numbers.parallelStream()
                   .peek(n -> System.out.println(Thread.currentThread().getName() + " processing " + n))
                   .mapToInt(Integer::intValue)
                   .sum()
        ).get(); // .submit() returns a ForkJoinTask/Future — get() blocks for the result

        System.out.println("Sum: " + sum);
        customPool.shutdown();
    }
}
```
**One possible output (worker names show it's the custom pool, not `ForkJoinPool.commonPool-worker-N`):**
```
ForkJoinPool-1-worker-1 processing 1
ForkJoinPool-1-worker-2 processing 3
ForkJoinPool-1-worker-3 processing 5
...
Sum: 36
```

> ⚠️ **Pitfall:** this is a genuine workaround, not officially blessed API — the JDK team has never guaranteed that a stream pipeline submitted this way won't ever spill work back onto the common pool in some edge cases (it reliably doesn't in current JDKs, but it's relying on `ForkJoinTask`'s ambient-pool behavior rather than a documented Streams-API contract). Mention this honestly if asked "is this guaranteed" — the honest senior answer is "it's the standard, working-in-practice technique, but it isn't a first-class Streams API feature," and for real isolation guarantees a manual `ExecutorService` + manually-partitioned tasks is the fully-supported alternative.

---

## Interview Q&A

**Q: Difference between Concurrency and Parallelism?**
Covered above.

**Q: How is the Fork/Join framework different from traditional thread pools?**
Covered above.

**Q: Explain ForkJoinPool's RecursiveTask specifically — how does fork()/join() work?**
Covered above.

**Q: RecursiveTask vs RecursiveAction — what's the difference?**
Covered above under "RecursiveAction" — `RecursiveTask<V>.compute()` returns `V`; `RecursiveAction.compute()` returns `void`. Everything else (fork/join mechanics, work-stealing, the compute-one-half-inline pattern) is identical.

**Q: How many threads does a ForkJoinPool use by default, and why is that different from ThreadPoolExecutor sizing?**
Covered above under "Sizing the ForkJoinPool" — defaults to one worker per CPU core, appropriate for CPU-bound divide-and-conquer work, unlike `ThreadPoolExecutor`'s I/O-bound sizing (Part 2). Never block on I/O inside a ForkJoinPool task, including the shared common pool.

**Q: How do you make a parallel stream run on a custom thread pool instead of the shared common pool?**
Covered above under "Custom Thread Pool for Parallel Streams" — wrap the pipeline in `customForkJoinPool.submit(() -> list.parallelStream()....).get()`. There's no direct `parallelStream(executor)` API; this relies on `ForkJoinTask` execution picking up the ambient pool of the submitting thread, which works reliably in practice but isn't a formally documented Streams API guarantee.
