# Part 10 — Async & Scheduling

> @Async gotchas, @Scheduled types, and configuring thread pools for both. Interview Q&A at the end.

## @Async — What It Does and the Gotchas in Getting It to Actually Run Asynchronously

```java
@EnableAsync  // on a @Configuration class — required, easy to forget
@Async
public CompletableFuture<String> processAsync() {
    return CompletableFuture.completedFuture("done");
}
```

**What it does:** runs the annotated method on a separate thread (from a configurable task executor) instead of the caller's thread. The method's declared return type must be `void`, `Future<T>`, or `CompletableFuture<T>` — anything else is a configuration error, since the caller can't get a synchronous return value from an async call.

**Why `@EnableAsync` is required:** it activates the underlying AOP proxy that intercepts `@Async`-annotated calls — without it, the annotation is silently ignored and the method just runs synchronously, no error.

> ⚠️ **Pitfall — same self-invocation bypass as `@Transactional`:** since `@Async` is also AOP-proxy-based, calling an `@Async` method via `this.asyncMethod()` from within the same bean skips the proxy entirely and runs synchronously, with no warning.

> ⚠️ **Pitfall — the default executor is dangerous:** the default task executor (`SimpleAsyncTaskExecutor`) creates a **new thread per task with no pooling and no bound** — fine for a demo, dangerous in production under load (unbounded thread creation). Always configure a proper `ThreadPoolTaskExecutor` bean explicitly for real workloads.

## @Scheduled vs @Async, the Three Scheduling Types, and Thread Pool Configuration

```java
@Configuration
@EnableScheduling
public class SchedulingConfig {
    @Bean
    public ThreadPoolTaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(10);
        scheduler.setThreadNamePrefix("ScheduledTask-");
        return scheduler;
    }
}
```

**The core distinction:** `@Async` runs a method asynchronously *when called* (one-off, on-demand); `@Scheduled` runs a method **periodically**, on its own schedule, with no external caller triggering it — different problems entirely despite both involving background execution.

**Three `@Scheduled` types:**
- `fixedRate` — starts every N ms **regardless** of whether the previous run finished — can overlap if a run takes longer than the interval.
- `fixedDelay` — waits N ms **after** the previous run completes before starting the next — never overlaps.
- `cron` — a full cron expression for calendar-based schedules.

**Requires `@EnableScheduling`.** By default, scheduled tasks run on a **single-threaded** scheduler, so multiple `@Scheduled` methods can block each other — define a `ThreadPoolTaskScheduler` bean with a real pool size for concurrent scheduled tasks.

> ⚠️ **Pitfall:** the `fixedRate`-can-overlap vs `fixedDelay`-never-overlaps distinction is the detail most often glossed over — if a `fixedRate` task's execution time exceeds its interval, you get concurrent overlapping runs unless the method itself is otherwise safe for that.

---

## Interview Q&A

**Q: What does @Async do, and what are the gotchas in getting it to actually run asynchronously?**
Covered above.

**Q: @Scheduled vs @Async, the three scheduling types, and configuring a thread pool for scheduled tasks?**
Covered above.
