# CompletableFuture — Complete Guide (Basic → Advanced)

> Every method explained in plain English, with a "when to use it" one-liner and a 2-line minimal example, plus a fuller runnable example with output. Read top to bottom for basic → advanced, or jump straight to the Quick Reference table if you already know roughly what you need.

![CompletableFuture chaining flow](../images/completablefuture-chain.svg)

## Quick Reference — "I need to... → use this"

| I need to... | Use | 2-line usage |
|---|---|---|
| Run something in the background and get a result later | `supplyAsync()` | `CompletableFuture<Integer> f = CompletableFuture.supplyAsync(() -> compute());`<br>`f.thenAccept(System.out::println);` |
| Run something in the background with no result | `runAsync()` | `CompletableFuture<Void> f = CompletableFuture.runAsync(() -> log("started"));`<br>`f.join();` |
| Transform a result into a new value | `thenApply()` | `CompletableFuture<Integer> f2 = f.thenApply(s -> s.length());`<br>`f2.thenAccept(System.out::println);` |
| Just consume a result (no new value) | `thenAccept()` | `f.thenAccept(result -> saveToDb(result));`<br>`// nothing chains after this` |
| Run something after completion, ignoring the result | `thenRun()` | `f.thenRun(() -> log.info("task finished"));`<br>`// no access to f's result at all` |
| Chain another **async call** that itself returns a future | `thenCompose()` | `CompletableFuture<String> f2 = getUserId().thenCompose(id -> getUserName(id));`<br>`f2.thenAccept(System.out::println);` |
| Combine 2 **independent** futures once both finish | `thenCombine()` | `CompletableFuture<Double> total = price.thenCombine(tax, (p, t) -> p + t);`<br>`total.thenAccept(System.out::println);` |
| Wait for **all** of N independent futures | `allOf()` | `CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2, f3);`<br>`all.thenRun(() -> System.out.println("all done"));` |
| React as soon as **any one** of N futures finishes | `anyOf()` | `CompletableFuture<Object> fastest = CompletableFuture.anyOf(f1, f2);`<br>`fastest.thenAccept(System.out::println);` |
| Recover from failure with a fallback value | `exceptionally()` | `f.exceptionally(ex -> -1);`<br>`// only runs if f failed` |
| React to success OR failure, and produce a new value either way | `handle()` | `f.handle((result, ex) -> ex != null ? -1 : result);`<br>`// runs regardless of outcome` |
| Do a side effect (logging) on success OR failure, without changing the result | `whenComplete()` | `f.whenComplete((result, ex) -> log(result, ex));`<br>`// exception still propagates afterward` |
| Run a callback on a specific thread pool, not the shared default | `...Async(fn, executor)` | `f.thenApplyAsync(x -> x * 2, myExecutor);`<br>`// avoids blocking the shared ForkJoinPool` |
| Block and get the result, checked-exception style | `get()` | `try { result = f.get(); }`<br>`catch (ExecutionException e) { ... }` |
| Block and get the result, unchecked-exception style | `join()` | `int result = f.join();`<br>`// throws unchecked CompletionException on failure` |
| Fall back to a default value if it takes too long | `completeOnTimeout()` | `f.completeOnTimeout("default", 500, TimeUnit.MILLISECONDS);`<br>`f.thenAccept(System.out::println);` |
| Treat "took too long" as a real failure | `orTimeout()` | `f.orTimeout(500, TimeUnit.MILLISECONDS);`<br>`f.exceptionally(ex -> "gave up");` |
| Wrap a value you already have, no async work needed | `completedFuture()` | `CompletableFuture<String> f = CompletableFuture.completedFuture("cached");`<br>`f.thenAccept(System.out::println);` |
| Manually complete a future from outside | `complete()` | `CompletableFuture<String> f = new CompletableFuture<>();`<br>`f.complete("value set from elsewhere");` |

---

## 1. Basics — What CompletableFuture Is

**What it does:** represents the result of an asynchronous computation that hasn't finished yet — similar to `Future`, but with the crucial addition that you can **attach callbacks** ("when this finishes, do X"), **chain** multiple async steps together, and **combine** multiple independent futures, all without ever blocking a thread just to wait.

**Why it exists over plain `Future`:** `Future.get()` is *blocking-only* — no way to attach a callback, no way to combine multiple futures, no way to complete it manually from outside. `CompletableFuture` adds all three.

### supplyAsync() — starting an async computation with a result

**When to use it:** you need to run something in the background that produces a value you'll use later.

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return "Result from async task";
});
future.thenAccept(result -> System.out.println("Received: " + result));
Thread.sleep(100); // give the async task time to finish before main exits
```
**Output:**
```
Received: Result from async task
```
By default this runs on `ForkJoinPool.commonPool()` — your calling thread does *not* wait for it.

### runAsync() — starting an async computation with no result

**When to use it:** fire-and-forget background work where you don't need a value back, just to know it ran.

```java
CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
    System.out.println("Thread: " + Thread.currentThread().getName());
    System.out.println("Hello from async task");
});
```
**Output:**
```
Thread: ForkJoinPool.commonPool-worker-1
Hello from async task
```

### completedFuture() — wrapping a value you already have

**When to use it:** some API expects a `CompletableFuture<T>`, but you don't actually need to do any async work — e.g. a cache hit, or a test stub.

```java
CompletableFuture<String> alreadyDone = CompletableFuture.completedFuture("cached-value");
alreadyDone.thenAccept(System.out::println);
```
**Output:**
```
cached-value
```

---

## 2. Reacting to a Result — thenApply, thenAccept, thenRun

These three are the ones people mix up most — the difference is exactly what each one **receives** and **returns**.

| Method | Receives previous result? | Returns a new value? | Use it when... |
|---|---|---|---|
| `thenApply(Function)` | Yes | Yes | you need to **transform** the result into something else, to keep chaining |
| `thenAccept(Consumer)` | Yes | No | you just need to **do something with** the result — print it, save it |
| `thenRun(Runnable)` | No | No | you just need to know the previous stage **finished**, regardless of what it produced |

### thenApply() — transform the result

**When to use it:** the next step needs to *produce* something new from the previous result, to feed further down the chain.

```java
CompletableFuture<Integer> future = CompletableFuture
    .supplyAsync(() -> 10)
    .thenApplyAsync(result -> result * 2);
System.out.println(future.get());
```
**Output:**
```
20
```

### thenAccept() — consume the result, produce nothing

**When to use it:** the chain effectively ends here — you want to act on the value (log, save, display) but don't need to transform it further.

```java
CompletableFuture.supplyAsync(() -> 10)
    .thenAccept(result -> System.out.println("Got: " + result));
```
**Output:**
```
Got: 10
```

### thenRun() — react to completion, ignoring the value entirely

**When to use it:** "now that this is done, go do this other unrelated thing" — e.g. logging "task finished" with no interest in what the task actually returned.

```java
CompletableFuture.supplyAsync(() -> 10)
    .thenRun(() -> System.out.println("Task finished"));
```
**Output:**
```
Task finished
```

---

## 3. Chaining Async Calls — thenCompose vs thenApply

**The problem thenCompose solves:** if your `thenApply` function itself returns a `CompletableFuture` (because it kicks off another async call), you end up with an ugly nested `CompletableFuture<CompletableFuture<R>>` — because `thenApply` just wraps whatever you return, and you returned a future.

**thenCompose(Function<T,CompletableFuture<R>>)** "flattens" the result instead of nesting — the same relationship as `map` vs `flatMap` on a Stream/Optional.

**When to use `thenCompose` instead of `thenApply`:** whenever the next step is itself another asynchronous call (another `supplyAsync`, another service call returning a `CompletableFuture`).

```java
static CompletableFuture<String> getUserId() {
    return CompletableFuture.supplyAsync(() -> "user123");
}
static CompletableFuture<String> getUserName(String userId) {
    return CompletableFuture.supplyAsync(() -> "Fetched name for " + userId);
}

public static void main(String[] args) throws Exception {
    // WRONG — nested: CompletableFuture<CompletableFuture<String>>
    CompletableFuture<CompletableFuture<String>> nested = getUserId().thenApply(id -> getUserName(id));

    // RIGHT — flattened: CompletableFuture<String>
    CompletableFuture<String> flattened = getUserId().thenCompose(id -> getUserName(id));
    System.out.println(flattened.get());
}
```
**Output:**
```
Fetched name for user123
```

---

## 4. Combining Multiple Futures — thenCombine, allOf, anyOf

### thenCombine() — merge exactly 2 independent futures

**When to use it:** two async operations don't depend on each other at all and can run in parallel, but you need **both** results together at the end.

```java
CompletableFuture<Integer> priceFuture = CompletableFuture.supplyAsync(() -> 100);
CompletableFuture<Double> taxRateFuture = CompletableFuture.supplyAsync(() -> 0.18);

CompletableFuture<Double> totalFuture = priceFuture.thenCombine(
    taxRateFuture,
    (price, taxRate) -> price + (price * taxRate)
);
System.out.println("Total: " + totalFuture.get());
```
**Output:**
```
Total: 118.0
```

### allOf() — wait for every one of N futures

**When to use it:** you're running several independent tasks in parallel and need to proceed only once **all** of them are done — the standard "fan out, then join" pattern.

```java
CompletableFuture<Void> task1 = CompletableFuture.runAsync(() -> System.out.println("Task 1"));
CompletableFuture<Void> task2 = CompletableFuture.runAsync(() -> System.out.println("Task 2"));
CompletableFuture<Void> combined = CompletableFuture.allOf(task1, task2);
combined.join();
System.out.println("Both done");
```
**One possible output:**
```
Task 1
Task 2
Both done
```
> ⚠️ **Pitfall:** `allOf()` returns `CompletableFuture<Void>` — it doesn't hand you the individual results directly. You still call `.join()` on each original future afterward to pull out its value (see the real-world example at the end of this guide).

### anyOf() — react to whichever finishes first

**When to use it:** "whichever responds first" patterns — e.g. querying two mirrored services and using whichever answers faster.

```java
CompletableFuture<String> serviceA = CompletableFuture.supplyAsync(() -> {
    try { Thread.sleep(200); } catch (InterruptedException e) {}
    return "Response from A";
});
CompletableFuture<String> serviceB = CompletableFuture.supplyAsync(() -> "Response from B");

CompletableFuture<Object> fastest = CompletableFuture.anyOf(serviceA, serviceB);
System.out.println(fastest.get());
```
**Output:**
```
Response from B
```

---

## 5. Exception Handling — exceptionally, handle, whenComplete

All three react to a stage finishing, but differ in what they can see and do:

| Method | Runs on success? | Runs on failure? | Can change the result? |
|---|---|---|---|
| `exceptionally(Function<Throwable,T>)` | No | Yes | Yes — supplies a fallback value |
| `handle(BiFunction<T,Throwable,R>)` | Yes | Yes | Yes — sees both, decides the outcome |
| `whenComplete(BiConsumer<T,Throwable>)` | Yes | Yes | No — side effect only, exception still propagates |

### exceptionally() — supply a fallback only on failure

**When to use it:** you want the chain to keep going with a sensible default instead of failing entirely, and you don't need to do anything special on the success path.

```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
    if (true) throw new RuntimeException("Something went wrong");
    return 10;
}).exceptionally(ex -> {
    System.out.println("Recovered from: " + ex.getMessage());
    return -1;
});
System.out.println("Final: " + future.get());
```
**Output:**
```
Recovered from: java.lang.RuntimeException: Something went wrong
Final: -1
```

### handle() — react to both outcomes and produce a new result

**When to use it:** you need one piece of logic that handles *both* "it worked" and "it failed" and decides what value comes out either way.

```java
CompletableFuture<Integer> future = CompletableFuture.<Integer>supplyAsync(() -> {
    throw new RuntimeException("failed");
}).handle((result, ex) -> {
    if (ex != null) {
        System.out.println("Handled failure: " + ex.getMessage());
        return -1;
    }
    return result;
});
System.out.println("Result: " + future.get());
```
**Output:**
```
Handled failure: java.lang.RuntimeException: failed
Result: -1
```

### whenComplete() — side effects only, doesn't swallow the exception

**When to use it:** logging or metrics that need to fire regardless of outcome, without changing what the caller ultimately sees (a failure still propagates as a failure).

```java
CompletableFuture<Integer> future = CompletableFuture.<Integer>supplyAsync(() -> 42)
    .whenComplete((result, ex) -> {
        if (ex != null) System.out.println("Logged failure: " + ex.getMessage());
        else System.out.println("Logged success: " + result);
    });
System.out.println(future.get());
```
**Output:**
```
Logged success: 42
42
```

---

## 6. Async Variants and Custom Executors

### The "Async" suffix — thenApplyAsync, thenAcceptAsync, etc.

**What it changes:** every chaining method has an `...Async` twin. Without it, the callback might run on whichever thread happened to complete the *previous* stage — which could even be your own calling thread, if the future was already done by the time you attached the callback. With `...Async`, the callback is guaranteed to run on a thread pool (the shared `ForkJoinPool.commonPool()` by default, or a custom one you supply) — giving predictable, decoupled execution.

```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> 10)
    .thenApplyAsync(result -> result * 2)
    .thenAcceptAsync(result -> System.out.println("Result: " + result));
future.get();
```
**Output:**
```
Result: 20
```
> ⚠️ **Pitfall:** relying on the non-Async variants makes your code's threading behavior inconsistent and hard to reason about, since it depends on timing. Prefer the `...Async` variants (with an explicit executor) whenever the exact execution thread matters.

### Custom Executor — don't block the shared pool

**When to use it:** any time your async work does **blocking I/O** (DB calls, HTTP calls) — running that on the default shared pool risks starving it for everyone else in your JVM (parallel streams and other libraries use the same pool).

```java
ExecutorService customExecutor = Executors.newFixedThreadPool(4);
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    System.out.println("Running on: " + Thread.currentThread().getName());
    return "done";
}, customExecutor);
System.out.println(future.get());
customExecutor.shutdown();
```
**Output:**
```
Running on: pool-1-thread-1
done
```
> ⚠️ **Pitfall:** always supply a dedicated `Executor` for I/O-bound async work in production code — a few slow blocking tasks on the default common pool can stall unrelated code across your whole JVM.

---

## 7. Blocking for the Result — get() vs join()

Both block the calling thread and return the result, but differ in how they report failure:

| | `get()` | `join()` |
|---|---|---|
| Checked exception | Throws checked `ExecutionException`/`InterruptedException` — must be caught or declared | Throws unchecked `CompletionException` — no forced handling |
| Common use | Traditional `Future`-style code | Stream pipelines / lambdas where checked exceptions are awkward |

```java
CompletableFuture<Integer> failed = CompletableFuture.supplyAsync(() -> { throw new RuntimeException("boom"); });
try {
    failed.get();
} catch (Exception e) {
    System.out.println("get() threw: " + e.getClass().getSimpleName());
}
try {
    failed.join();
} catch (Exception e) {
    System.out.println("join() threw: " + e.getClass().getSimpleName());
}
```
**Output:**
```
get() threw: ExecutionException
join() threw: CompletionException
```

---

## 8. Timeouts — orTimeout() and completeOnTimeout() (Java 9+)

Both put a time limit on how long you'll wait, but react differently when the deadline passes.

**completeOnTimeout(fallback, ...)** — when to use it: you'd rather silently substitute a sensible default than fail the whole chain.
```java
CompletableFuture<String> slow = CompletableFuture.supplyAsync(() -> {
    try { Thread.sleep(2000); } catch (InterruptedException e) {}
    return "slow result";
});
CompletableFuture<String> withFallback = slow.completeOnTimeout("fallback value", 500, TimeUnit.MILLISECONDS);
System.out.println(withFallback.get());
```
**Output:**
```
fallback value
```

**orTimeout(...)** — when to use it: a timeout should be treated as a genuine failure that needs handling, not silently papered over.
```java
slow.orTimeout(500, TimeUnit.MILLISECONDS)
    .exceptionally(ex -> "gave up: " + ex.getClass().getSimpleName());
```

---

## 9. Manual Completion — complete(), completeExceptionally(), cancel()

**When to use these:** bridging a non-`CompletableFuture`-based async API (e.g. a callback-style library) into the `CompletableFuture` world — you create an empty future, and complete it yourself once the callback fires.

```java
CompletableFuture<String> future = new CompletableFuture<>();

// somewhere else, e.g. inside a callback:
future.complete("value set from elsewhere");
// or, on failure: future.completeExceptionally(new RuntimeException("failed"));

future.thenAccept(System.out::println);
```
**Output:**
```
value set from elsewhere
```
Unlike a plain `Future` (which you can only wait on after submitting to an executor), a `CompletableFuture` can be completed from **anywhere**, at any time — this manual-completion ability is one of its defining advantages over `Future`.

---

## 10. Real-World Use Case — Parallel Microservice Calls

A common production pattern: call 3 independent downstream services in parallel instead of one after another, then combine their results once all three return.

```java
ExecutorService executor = Executors.newFixedThreadPool(3);

CompletableFuture<String> userServiceCall = CompletableFuture.supplyAsync(() -> {
    return "User: Sabari";
}, executor);

CompletableFuture<String> orderServiceCall = CompletableFuture.supplyAsync(() -> {
    return "Orders: 5";
}, executor);

CompletableFuture<String> paymentServiceCall = CompletableFuture.supplyAsync(() -> {
    return "Payments: $250";
}, executor);

CompletableFuture<Void> allCalls = CompletableFuture.allOf(userServiceCall, orderServiceCall, paymentServiceCall);

CompletableFuture<String> combinedResponse = allCalls.thenApply(v ->
    userServiceCall.join() + " | " + orderServiceCall.join() + " | " + paymentServiceCall.join()
);

System.out.println(combinedResponse.get());
executor.shutdown();
```
**Output:**
```
User: Sabari | Orders: 5 | Payments: $250
```
> Sequentially, this takes `t1 + t2 + t3` total time; running in parallel via `allOf()` takes roughly `max(t1, t2, t3)` — the entire point of using `CompletableFuture` for I/O-bound work.

---

## Interview Q&A

**Q: Future vs CompletableFuture — what's the difference?**
`Future.get()` blocks with no way to attach a callback, combine futures, or complete manually. `CompletableFuture` adds non-blocking completion handling (`thenApply`, `thenAccept`), fluent composition (`thenCompose`, `thenCombine`, `allOf`), and manual completion (`complete()`/`completeExceptionally()`).
> ⚠️ **Pitfall:** "CompletableFuture doesn't block" is only true if you use its callback methods — calling `.get()` on one still blocks exactly like a plain `Future`.

**Q: thenApply vs thenAccept vs thenRun — what's the difference?**
Covered above in section 2 — the short version: `thenApply` transforms and returns a value, `thenAccept` consumes the value and returns nothing, `thenRun` doesn't even see the value.
> ⚠️ **Pitfall:** using `thenApply` just to throw away its return value (when `thenAccept` would do) is a minor but real code smell — it signals you didn't think about which one actually fits.

**Q: How do you run multiple CompletableFutures in parallel and wait for all of them?**
`CompletableFuture.allOf(f1, f2, f3)` — see section 4 and the real-world example above.

**Q: How do you handle errors in a CompletableFuture chain?**
`exceptionally()` for a simple fallback value, `handle()` when you need one function covering both success and failure, `whenComplete()` for side-effect-only logging that doesn't swallow the original exception. See section 5.
