# Exception Handling

> The full exception model from the ground up: hierarchy and `Throwable` internals, exact `try/catch/finally` mechanics, try-with-resources, designing custom exceptions, the overriding rules, production debugging (OOM variants, heap dumps, NPEs), Spring/stream/concurrency integration points, and how exceptions actually work at the JVM bytecode level. Interview Q&A at the end.

## The Hierarchy, and What `Throwable` Actually Stores

```
java.lang.Throwable
├── java.lang.Error                          -- JVM/environment failures, not application logic
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   ├── NoClassDefFoundError
│   ├── AssertionError
│   └── VirtualMachineError
└── java.lang.Exception
    ├── RuntimeException (unchecked)         -- programming errors, compiler doesn't enforce handling
    │   ├── NullPointerException
    │   ├── ArrayIndexOutOfBoundsException
    │   ├── ClassCastException
    │   ├── IllegalArgumentException
    │   ├── IllegalStateException
    │   └── ConcurrentModificationException
    └── Checked Exceptions                   -- compiler forces you to catch or declare
        ├── IOException → FileNotFoundException, SocketException
        ├── SQLException
        ├── ClassNotFoundException
        └── InterruptedException
```

**What every `Throwable` carries as internal state:** `detailMessage` (the message string), `cause` (the wrapped original exception — the chaining mechanism), `stackTrace` (an array of `StackTraceElement`, captured at *construction* time, not throw time), and `suppressedExceptions` (populated by try-with-resources, Java 7+).

**Why creating an exception is expensive:** the `Throwable` constructor calls the native method `fillInStackTrace()`, which walks the *entire current call stack* right there at construction — in a deep layered Spring app (controller → service → repository → ...), that's real, non-trivial work. This single fact is the root cause of two other rules later in this doc: why exceptions must never be used for routine control flow, and why the *actual* expensive part of "throwing an exception" is capturing the trace, not the catch-block lookup itself.

## Errors vs Exceptions — the Real Distinction

**Errors** represent conditions the *JVM itself* can't recover from — environment/runtime failures, not application logic (`OutOfMemoryError`, `StackOverflowError`, `NoClassDefFoundError`). **Exceptions** represent conditions the *application* can reasonably anticipate and potentially recover from.

Technically you *can* `catch (OutOfMemoryError e)` — it compiles and runs, since `Error extends Throwable`. You almost never should:
1. The JVM may already be in an unstable state, with more failures likely.
2. For OOM specifically, the catch block's own allocations can throw a second OOM.
3. Catching broadly hides catastrophic failures from monitoring that should be alerting on them.

> ⚠️ **Pitfall:** the one legitimate exception is catching `OutOfMemoryError` at the very top level *specifically* to log and trigger a graceful shutdown — some enterprise frameworks do this deliberately as a last resort, but it's not general practice.

## Checked vs Unchecked — the Actual Design Guidance

Checked exceptions force callers to consider failure, at the cost of boilerplate. Unchecked exceptions give a cleaner API but let callers silently ignore failure modes. The precise guidance (*Effective Java*, Item 71) is sharper than "always use unchecked": **use checked exceptions when a caller can reasonably recover** — retry, fall back, prompt the user — **use unchecked for programming errors** (an unexpected null, an invalid argument) the immediate caller usually can't meaningfully act on. Most modern Spring codebases lean unchecked-by-default specifically because checked exceptions on every method pollute every caller's signature across a deep call chain.

## `throw` vs `throws`, and the Three Ways to Rethrow

```java
void process(String input) throws IOException {
    if (input == null) {
        throw new IllegalArgumentException("Input must not be null"); // throw -- actually raises it, in the body
    }
    readFile(input); // may throw IOException, declared via throws in the signature
}
```
A method *can* declare more exceptions in `throws` than it actually ever throws — legal, but bad practice, since it forces every caller to handle failure modes that can never occur.

**Does `throw null;` compile?** Yes — but it throws `NullPointerException` at runtime; the JVM checks the throw target is non-null immediately before the throw happens.

**Three ways to rethrow:**
```java
catch (IOException e) { throw e; }                                    // 1. same exception, unchanged
catch (IOException e) { throw new ServiceException("Failed", e); }    // 2. wrapped/chained -- the default pattern
catch (Exception e) { throw e; }  // only ONE type actually thrown inside try -- precise rethrow (Java 7+): compiler infers the real type
```

## The Exact Mechanics of `try/catch/finally`

**When does `finally` get skipped?** Exactly four cases: `System.exit()` inside the `try`; a fatal JVM crash; an infinite loop/deadlock inside `try` (control never leaves it); the OS kills the process externally (`kill -9`).

**What happens when `finally` itself throws?** It silently destroys whatever exception was already propagating:
```java
try {
    throw new IOException("original");
} finally {
    throw new RuntimeException("from finally"); // the IOException is GONE -- not chained, not suppressed, permanently lost
}
// caller sees ONLY the RuntimeException
```

**What happens when `finally` contains a `return`?** It silently overrides the `try` block's return value:
```java
int compute() {
    try { return 1; } finally { return 2; } // caller receives 2, not 1
}
```
> ⚠️ **Pitfall:** both of the above are why `finally` should be used **only for cleanup**, never for control flow (`return`/`throw`). The correct resource-cleanup pattern:
```java
Connection conn = null;
try {
    conn = getConnection();
    return doWork(conn);
} finally {
    if (conn != null) conn.close(); // cleanup only -- no return, no throw
}
```

## Multi-Catch and Catch Ordering

```java
catch (IOException | SQLException ex) {
    log(ex); // ex is implicitly final -- cannot be reassigned
}
```
Multi-catch types cannot be in a parent-child relationship — `catch (Exception | IOException ex)` is a compile error, since `IOException` already *is* an `Exception`.

**Catch blocks must be ordered specific-to-general** — listing a broader type before a narrower one is a compile error (`unreachable catch block`), not just a style nit. This maps directly onto how the JVM actually resolves exceptions at runtime (see the exception table section below) — table entries are matched in order, so an out-of-order broad entry would silently shadow the narrower one beneath it.

## try-with-resources — What It Actually Compiles Into

```java
BufferedReader br = new BufferedReader(new FileReader("file.txt"));
Throwable primaryException = null;
try {
    // try body
} catch (Throwable t) {
    primaryException = t;
    throw t;
} finally {
    if (br != null) {
        if (primaryException != null) {
            try { br.close(); }
            catch (Throwable suppressed) { primaryException.addSuppressed(suppressed); } // the key mechanism
        } else {
            br.close();
        }
    }
}
```
**What happens when both the try body and `close()` throw?** The `close()` exception is **not lost** — it's attached via `addSuppressed()`, retrievable later via `e.getSuppressed()`. This is precisely the fix for the "exception from `finally` destroys the original" failure mode above — try-with-resources exists specifically to avoid that.

**Multiple resources close in reverse declaration order (LIFO):**
```java
try (Connection conn = getConn();
     PreparedStatement stmt = conn.prepareStatement(sql);
     ResultSet rs = stmt.executeQuery()) {
    // use rs
}
// closed: rs -> stmt -> conn
```
**Java 9+:** effectively-final variables declared *outside* the `try` can be referenced directly, without redeclaring: `try (conn) { ... }`.

## Designing Custom Exceptions

**A well-formed custom exception** — unchecked, constructors matching `RuntimeException` conventions, and critically, always providing a cause-accepting constructor so chaining survives:
```java
public class ProductNotFoundException extends RuntimeException {
    private final String productId;
    public ProductNotFoundException(String message) { super(message); this.productId = null; }
    public ProductNotFoundException(String productId, String message) { super(message); this.productId = productId; }
    public ProductNotFoundException(String message, Throwable cause) { super(message, cause); this.productId = null; } // never skip this
    public String getProductId() { return productId; }
}
```
**An enterprise error-code hierarchy** — scales far better once you have dozens of exception types needing consistent client-facing error responses:
```java
public class AppException extends RuntimeException {
    private final ErrorCode errorCode;
    public enum ErrorCode {
        PRODUCT_NOT_FOUND("PROD-001", "Product not found"),
        PAYMENT_FAILED("PAY-001", "Payment processing failed");
        private final String code; private final String defaultMessage;
        ErrorCode(String code, String defaultMessage) { this.code = code; this.defaultMessage = defaultMessage; }
        public String getDefaultMessage() { return defaultMessage; }
    }
    public AppException(ErrorCode errorCode, String details, Throwable cause) {
        super(errorCode.getDefaultMessage() + ": " + details, cause);
        this.errorCode = errorCode;
    }
}
```
**Decision framework:** checked when the caller can realistically recover (a timeout worth retrying); unchecked for programming errors, or when designing a library API where checked exceptions on every method would pollute every caller.

> ⚠️ **Pitfall — exceptions as control flow:**
```java
// BAD -- pays fillInStackTrace() cost just to branch on a parse failure
try { int value = Integer.parseInt(input); } catch (NumberFormatException e) { value = 0; }
// GOOD -- explicit check, no exception machinery at all
int value = input.matches("\\d+") ? Integer.parseInt(input) : 0;
```

## Overriding Rules for Checked Exceptions

An overriding method **cannot** throw new or broader checked exceptions than the parent declares — it can throw the same, a narrower one, or none. Unchecked exceptions are entirely unrestricted.
```java
class Parent { void show() throws IOException { } }
class Child extends Parent {
    void show() throws IOException { }            // same -- OK
    void show() throws FileNotFoundException { }  // narrower -- OK
    void show() { }                                // none -- OK
    void show() throws Exception { }               // broader -- COMPILE ERROR
    void show() throws SQLException { }             // new, unrelated checked -- COMPILE ERROR
}
```
**Why:** direct consequence of polymorphism / Liskov substitution — `Parent p = new Child(); p.show();` — the caller only knows `Parent`'s declared exceptions; if `Child` could throw something broader, calling code prepared only for `Parent`'s contract would be caught off guard by a type it never accounted for. If `Parent` declares *no* checked exception at all, `Child` can't introduce one either. A child **can** legally drop a `throws` entirely — that's a strictly narrower, safe contract.

## `NoClassDefFoundError` vs `ClassNotFoundException`

`ClassNotFoundException` is **checked**, thrown by application-level APIs (`Class.forName()`, `ClassLoader.loadClass()`) when a class genuinely can't be located during an *explicit* dynamic load. `NoClassDefFoundError` is an `Error` (`LinkageError`), thrown by the JVM itself when a class that *was* available and verified at compile time is missing at runtime while the JVM is implicitly resolving a reference.

**Failed static initializer cascade:** the **first** access throws `ExceptionInInitializerError`; **every subsequent** attempt to use that class in that classloader throws `NoClassDefFoundError` — the JVM marks the class permanently broken after one failed init.
```java
class Broken { static { if (true) throw new RuntimeException("init failed!"); } }
// 1st access: ExceptionInInitializerError
// every access after: NoClassDefFoundError
```
> ⚠️ **Pitfall — the "it worked yesterday" playbook:** check for a recent deployment removing/changing a JAR, check the classpath itself, and scroll **up** in the logs above the `NoClassDefFoundError` for an earlier `ExceptionInInitializerError` — that's very often the real root cause hiding above the symptom. `java -verbose:class` and `jdeps` help confirm what's actually loading.

## Production Debugging — OutOfMemoryError

**The variants have completely different root causes** — knowing which one you have changes the entire investigation:
- `Java heap space` — heap full, objects can't be collected (real leak, or genuinely oversized working set).
- `GC overhead limit exceeded` — JVM spending >98% of time in GC reclaiming <2% of heap.
- `Metaspace` — class metadata exhausted, typically from excessive dynamic class generation (CGLIB proxies, Groovy).
- `Direct buffer memory` — `ByteBuffer.allocateDirect()` pool exhausted; common with NIO/Netty/Kafka clients.
- `Unable to create new native thread` — OS thread limit hit, usually unbounded thread creation rather than pooling.

**Step one after an OOM alert: capture diagnostics before restarting** — restarting destroys the evidence.
```
jmap -dump:format=b,file=heap.hprof <pid>
jstack <pid> > threads.txt
```
Ideally automated proactively: `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log/app/heapdump.hprof`

**Analyzing the dump:** Eclipse Memory Analyzer (MAT) — the **Dominator Tree** shows which objects retain the most memory, the **Leak Suspects Report** auto-detects likely culprits, **OQL** lets you query the heap directly.

**Most common leak patterns to check first:** a `static` collection used as an unbounded cache; loading an entire table into memory (`repo.findAll()` on a huge table); HTTP sessions never invalidated; `ThreadLocal.set()` without a matching `remove()` in a pooled-thread environment (the thread — and the leaked value — outlives any individual request); classloader leaks from hot-redeploy app servers; an unbounded Kafka consumer queue where commits can't keep pace with intake.

**The Kubernetes-specific gotcha:** `-Xmx` must be set **below**, not equal to, the container memory limit — the JVM needs headroom for metaspace, thread stacks, native memory. Rule of thumb: `-Xmx` ≈ 75% of the container limit, with `-XX:+UseContainerSupport` (default since Java 10) so the JVM correctly reads cgroup limits. An `OOMKilled` pod status is the **OS/cgroup** killing the container for exceeding its limit — a different failure from a Java-level `OutOfMemoryError`; `kubectl describe pod` tells you which one actually happened.

**Confirming a real leak via GC logs:** enable with `-Xlog:gc*:file=/var/log/gc.log:time,uptime,level,tags:filecount=10,filesize=20m`. The signal: heap usage **after full GC** should stay roughly stable over time. A steadily climbing post-GC baseline (200MB → 300MB → 450MB) is the clearest signature of a real leak, versus a temporarily busy period that a full GC successfully reclaims back down.

## Production Debugging — NullPointerException

**Playbook for an NPE that only happens in production:** first hypothesize the dev/prod mismatch — production data with nulls dev's seed data doesn't have; a prod-only config resolving to null; a race condition only manifesting under real load; an external service returning null under a timeout that never happens against a stable dev mock. Then: read the stack trace precisely, check for a specific triggering input/ID, pull live diagnostics without redeploying (Actuator log-level changes, distributed tracing), grab a thread dump if concurrency is suspected, and apply defensive fixes:
```java
Objects.requireNonNull(user, "User must not be null for operation X");
Optional<User> user = userRepository.findById(id);
user.orElseThrow(() -> new UserNotFoundException("User " + id + " not found"));
```
**Java 14+ helpful NPE messages** (default since Java 15, `-XX:+ShowCodeDetailsInExceptionMessages`) name the exact null variable:
```
Old:  NullPointerException at UserService.java:42
New:  Cannot invoke "User.getName()" because "user" is null at UserService.java:42
```
Worth confirming this flag is actually enabled on older deployments.

## Spring Integration

**How `@ControllerAdvice` works internally:** scanned/registered as a Spring bean at startup, wired into `ExceptionHandlerExceptionResolver` — one of the `HandlerExceptionResolver`s `DispatcherServlet` chains through when a controller throws.
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(ProductNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleProductNotFound(ProductNotFoundException ex, HttpServletRequest req) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("PROD-001", ex.getMessage(), req.getRequestURI()));
    }
    @ExceptionHandler(Exception.class) // last-resort catch-all
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex, HttpServletRequest req) {
        log.error("Unexpected error at {}", req.getRequestURI(), ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("SYS-001", "An unexpected error occurred", req.getRequestURI()));
    }
}
```
**Resolution priority:** the most specific exception type wins; an `@ExceptionHandler` defined directly inside the throwing controller beats a global `@ControllerAdvice` handler, even if the advice targets a more specific type.

> ⚠️ **Pitfall — three places `@ControllerAdvice` does NOT catch anything:** (1) **Servlet `Filter`s** — run before `DispatcherServlet` is even involved; need their own try-catch. (2) **`@Async` methods** — run on a separate thread, outside the request's exception path entirely; need a dedicated `AsyncUncaughtExceptionHandler` (see below). (3) **Spring Security filters** — auth failures happen in the security filter chain, ahead of `DispatcherServlet`; handled via `AuthenticationEntryPoint`/`AccessDeniedHandler`. Teams that only wire up `@ControllerAdvice` typically discover these gaps the hard way, as an unformatted stack trace leaking straight to a client.

**Does `@Transactional` roll back on checked exceptions by default?** No — by default Spring rolls back only on unchecked `RuntimeException`/`Error`, not checked exceptions, regardless of how serious they conceptually are. Override with `@Transactional(rollbackFor = SomeCheckedException.class)`.

**Lombok's `@SneakyThrows`:** uses bytecode manipulation to throw a checked exception without declaring it in `throws` — tricking only `javac`'s compile-time check (the JVM itself never enforced checked-exception declarations). Genuinely controversial: some teams ban it as breaking the checked-exception contract; others use it deliberately to cut boilerplate.

## Checked Exceptions Inside Streams

`Function`/`Predicate`/`Consumer` declare no checked exceptions in their abstract methods, so a checked-exception-throwing call can't sit directly inside a lambda passed to `.map()`. Three real patterns:
```java
// 1. Optional wrapper -- "skip and continue on individual failures"
private Optional<Integer> safeParseInt(String s) {
    try { return Optional.of(Integer.parseInt(s)); } catch (NumberFormatException e) { return Optional.empty(); }
}

// 2. checked-to-unchecked wrapper -- "this should never fail, treat it as a bug if it does"
@FunctionalInterface interface CheckedFunction<T, R> { R apply(T t) throws Exception; }
static <T, R> Function<T, R> wrap(CheckedFunction<T, R> fn) {
    return t -> { try { return fn.apply(t); } catch (Exception e) { throw new RuntimeException(e); } };
}

// 3. partition successes/failures -- process everything, report both outcomes
Map<Boolean, List<String>> partitioned = items.stream()
    .collect(Collectors.partitioningBy(item -> { try { process(item); return true; } catch (Exception e) { return false; } }));
```

## Exception Chaining — the One Non-Negotiable Rule

```java
// BAD -- the original SQLException and its whole stack trace is gone forever
catch (SQLException e) { throw new ServiceException("DB error"); }
// GOOD -- cause is preserved, appears in the full chain
catch (SQLException e) { throw new ServiceException("Failed to load user", e); }
```
**The most common violation in real codebases** is exactly the "BAD" version above — a "lost cause," where the log shows only the symptom with zero indication of what actually failed underneath. **Swallowing `InterruptedException`** is a related, specifically dangerous version of this:
```java
catch (InterruptedException e) { /* ignore */ }                                       // BAD -- interrupted status permanently lost
catch (InterruptedException e) { Thread.currentThread().interrupt(); throw new RuntimeException(e); } // GOOD -- restore the flag
```
`ExecutorService` shutdown logic and other cooperative-cancellation code (Part 2 of the multithreading notes) depends on that interrupt flag actually being set — silently swallowing it breaks graceful shutdown elsewhere in the system that has nothing obviously to do with where you swallowed it.

## Exceptions and Concurrency

**`execute(Runnable)` vs `submit()` — where does the exception actually go?** An exception in `execute()` propagates to the pool's worker thread, killing it — the uncaught exception handler fires (default: `System.err`), and the pool silently replaces the dead worker. **No direct failure signal** unless you're watching stderr. An exception in `submit()` is caught internally and stored inside the returned `Future` — the worker survives — and only resurfaces (wrapped in `ExecutionException`) when you call `future.get()`.

> ⚠️ **Pitfall — the dangerous trap:** call `submit()` and never call `.get()` (common in fire-and-forget code), and the exception is captured, stored, and **silently discarded forever** — no log line, nothing. Counterintuitively, for true fire-and-forget work, `execute()` is *safer* than an ignored `submit()`, precisely because it doesn't hide the failure. If using `submit()`, always consume the `Future`.

**Exception propagation through `CompletableFuture`:** short-circuits every subsequent `thenApply`/`thenAccept`/`thenRun` stage, jumping straight to the nearest exception-handling stage:
```java
CompletableFuture<Integer> result = CompletableFuture.supplyAsync(() -> riskyOperation()) // throws here
    .thenApply(x -> x * 2)   // SKIPPED
    .thenApply(x -> x + 1)   // SKIPPED
    .exceptionally(ex -> { log.error("Pipeline failed", ex); return -1; }); // recovery value, pipeline continues
```
`exceptionally` only fires on failure (like `catch`); `handle` runs unconditionally and can transform the result either way; `whenComplete` runs unconditionally but is side-effect-only — it **cannot** change the outcome, the exception is still propagating after it runs.

> ⚠️ **Pitfall — the silent `instanceof` failure:** exceptions from `*Async` stages get wrapped in `CompletionException` before reaching `exceptionally`/`handle`. `if (ex instanceof MyCustomException)` never matches; you need `ex.getCause() instanceof MyCustomException`. Relatedly, `get()` throws checked `ExecutionException` while `join()` throws unchecked `CompletionException` — same underlying cause, different wrapper depending purely on which terminal call you used.

**`AsyncUncaughtExceptionHandler` only fires for `void`-returning `@Async` methods.** If the method returns a `Future`/`CompletableFuture`, the handler is **never invoked** — the exception is captured in the returned future exactly like the `submit()` case above, and it's entirely the caller's job to `.get()` it or chain `.exceptionally()`. Also: a saturated `@Async` thread pool's queue can throw `TaskRejectedException` **synchronously on the calling thread**, at invocation time — the one failure mode here needing an ordinary try-catch rather than `exceptionally()`. And `SecurityContext` does **not** auto-propagate into `@Async` threads without an explicit `TaskDecorator` — an auth check failing inside `@Async` code that assumed inherited context is a classic confusing-only-when-async bug.

## How Exceptions Actually Work at the JVM Level

**The exception table:** the compiler emits a table alongside each method's bytecode — each entry has `start_pc`/`end_pc` (the bytecode range covered), `handler_pc` (jump target), and `catch_type`.
```
Exception table:
   from    to  target  type
     0     8      11   java/io/IOException
     0     8      20   java/sql/SQLException
```
**Finding the right handler:** there's no "searching through nested try blocks" the way source-level nesting visually suggests. On a throw, the JVM checks the *current method's* table for an entry covering the current program counter whose `catch_type` matches — checked **in table order**, exactly why catch ordering must be specific-first (see above). No match → the JVM **pops the entire current stack frame** and repeats the lookup in the caller's frame. That frame-popping repeat is **stack unwinding**.

**How `finally` compiles**, given it isn't a real bytecode construct: modern `javac` adds an extra catch-all table entry (`catch_type` = "matches anything") covering the try/catch region, whose handler runs the `finally` code and then **rethrows** whatever was caught — guaranteeing `finally` runs on the exceptional path without literally duplicating the bytecode at every exit (which is what older compilers actually did).

> ⚠️ **Pitfall — why exception-based control flow is expensive, precisely:** the table lookup itself is fast; it's genuinely not the bottleneck. The real cost is `fillInStackTrace()` running inside the `Throwable` constructor *before* the throw even begins propagating — a native call walking every stack frame, paid regardless of how cheap the subsequent table-based unwinding is. People assume unwinding is the slow part; it's actually the earlier trace-capture step.

**Does the JVM guarantee locks are released during unwinding?** For `synchronized` — **yes**, structurally: the `monitorenter` bytecode is always paired with both a normal-path and an exception-path `monitorexit`, via an implicit exception-table entry covering the block. For `ReentrantLock` — **no**. `unlock()` is just an ordinary method call; unwinding past it without an explicit `finally` leaves the lock held forever:
```java
lock.lock();
riskyCall(); // if this throws with no finally, the lock is held FOREVER
lock.unlock();
```
This is exactly why `ReentrantLock` needs the `try { } finally { lock.unlock(); }` discipline that `synchronized` gets automatically at the JVM level — a direct callback to Part 4 of the multithreading notes.

> ⚠️ **Pitfall — cleanup is not guaranteed in general:** only `synchronized` monitors get this structural JVM-level protection. Everything else (file handles, DB connections, explicit locks) is cleaned up only if a programmer wired it into `finally`/try-with-resources — and even that can be defeated: during a `StackOverflowError`, the stack is already exhausted, and if a `finally` block itself does anything stack-intensive, it can throw a *second* `StackOverflowError` mid-cleanup, defeating the original cleanup entirely. "Gracefully recover from `StackOverflowError`" deserves the same skepticism as recovering from `OutOfMemoryError`.

---

## Interview Q&A — Rapid Fire

**Q: Why is throwing an exception expensive, and is it the `catch` lookup that's slow?**
No — the exception-table lookup is fast. The expense is `fillInStackTrace()` in the `Throwable` constructor walking the entire call stack at *construction* time, before the throw even propagates.

**Q: `finally` contains a `return` — what happens to the `try` block's return value?**
It's silently discarded; the `finally` block's `return` wins. Same failure category as an exception thrown inside `finally` destroying whatever was already propagating — `finally` should only ever do cleanup, never control flow.

**Q: How does try-with-resources avoid the "exception in finally destroys the original" problem?**
It uses `addSuppressed()` — if both the try body and `close()` throw, the `close()` exception is attached to the primary one (retrievable via `getSuppressed()`), never silently lost.

**Q: `execute()` vs `submit()` on an `ExecutorService` — where do their exceptions go, and which is actually safer for fire-and-forget work?**
`execute()`'s exception kills the worker thread and fires the uncaught-exception handler (visible on stderr); `submit()`'s exception is captured inside the `Future` and surfaces only via `.get()`. Counterintuitively `execute()` is safer for pure fire-and-forget work, because an ignored `submit()` silently swallows the failure forever.

**Q: Does `synchronized` guarantee lock release if the block throws? Does `ReentrantLock`?**
`synchronized` — yes, structurally, via the JVM's exception-table handling of `monitorenter`/`monitorexit`. `ReentrantLock.unlock()` is an ordinary method call with no such guarantee — it requires an explicit `finally` or the lock is held forever.

**Q: Your `NoClassDefFoundError` started appearing after a deployment that changed nothing you can find — what's the first thing to check?**
Scroll up in the logs for an earlier `ExceptionInInitializerError` on that same class — a static initializer failing once permanently marks the class unusable, and every access after the first throws `NoClassDefFoundError` with no obvious link back to the real root cause.
