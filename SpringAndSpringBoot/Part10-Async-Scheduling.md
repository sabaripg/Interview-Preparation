# ⏰ Part 10 — Async & Scheduling

> Neat, point-based format with callout boxes, tables, and icons. Interview Q&A at the end.

---

## ⚡ @Async — What It Does and the Gotchas

```java
@EnableAsync  // on a @Configuration class — required, easy to forget
@Async
public CompletableFuture<String> processAsync() {
    return CompletableFuture.completedFuture("done");
}
```

- Runs the annotated method on a separate thread instead of the caller's.
- Return type must be `void`, `Future<T>`, or `CompletableFuture<T>` — anything else is a configuration error.
- Requires `@EnableAsync` to activate the AOP proxy — without it, the annotation is **silently ignored** and the method runs synchronously, no error.

> [!CAUTION]
> Same **self-invocation bypass** as `@Transactional` (Part 5) — since `@Async` is also AOP-proxy-based, calling it via `this.asyncMethod()` from the same bean skips the proxy and runs synchronously, no warning.

> [!WARNING]
> The default task executor (`SimpleAsyncTaskExecutor`) creates a **new thread per task with no pooling and no bound** — dangerous in production under load. Always configure a proper `ThreadPoolTaskExecutor` bean explicitly.

---

## 📅 @Scheduled vs @Async, the Three Scheduling Types

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

| | `@Async` | `@Scheduled` |
|---|---|---|
| Trigger | Called on-demand, one-off | Runs periodically, on its own schedule, no caller |

**The three scheduling types:**

| Type | Behavior | Can overlap? |
|---|---|---|
| `fixedRate` | Starts every N ms regardless of previous run's completion | ✅ Yes, if a run exceeds the interval |
| `fixedDelay` | Waits N ms **after** the previous run completes | ❌ Never |
| `cron` | Full cron expression for calendar-based schedules | Depends on expression |

> [!IMPORTANT]
> Requires `@EnableScheduling`. By default, scheduled tasks run on a **single-threaded** scheduler, so multiple `@Scheduled` methods can block each other — define a `ThreadPoolTaskScheduler` bean for real concurrency.

---

## 📋 Interview Q&A

| Question | Short answer |
|---|---|
| What does @Async do, and its gotchas? | Runs on a separate thread; needs @EnableAsync; self-invocation bypass; default executor is unbounded |
| @Scheduled vs @Async, the three types, thread pool config? | On-demand vs periodic; fixedRate can overlap, fixedDelay never does |
