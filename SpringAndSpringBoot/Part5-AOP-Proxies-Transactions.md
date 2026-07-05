# Part 5 — AOP, Proxies & Transactions

> AOP fundamentals, custom annotations, @Transactional internals, propagation types, and every silent-failure gotcha they share a root cause with. Interview Q&A at the end.

## AOP Fundamentals — Aspect, Advice, Pointcut, Join Point, Weaving

| Term | Meaning |
|---|---|
| **Aspect** | Module encapsulating cross-cutting logic — the `@Aspect`-annotated class |
| **Advice** | The action taken — `@Before`, `@After`, `@Around`, `@AfterReturning`, `@AfterThrowing` |
| **Pointcut** | An expression selecting WHICH join points get advised |
| **Join Point** | A point where advice CAN apply — in Spring AOP, always a method call |
| **Weaving** | Linking aspects into the target — Spring does this at RUNTIME via proxies (not compile-time, unlike full AspectJ) |

```java
@Aspect
@Component
public class LoggingAspect {
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {}

    @Around("serviceLayer()")
    public Object logExecutionTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed(); // MUST call this or the real method never runs
        log.info("{} took {}ms", pjp.getSignature().getName(), System.currentTimeMillis() - start);
        return result;
    }
}
```

> ⚠️ **Pitfall:** `@Around` fully replaces the method's execution flow — forgetting `pjp.proceed()` means the real method silently never runs at all, a no-op bug with no exception thrown:
```java
@Around("serviceLayer()")
public Object broken(ProceedingJoinPoint pjp) throws Throwable {
    log.info("Before");
    // FORGOT: pjp.proceed();
    return null; // real method NEVER executes
}
```
This underlying proxy mechanism is exactly why `final` classes/methods and `private` methods silently break `@Transactional`/`@Async`/`@Cacheable` too — CGLIB needs to subclass and override, which `final`/`private` block. All of these "silent AOP failures" share one root cause.

## Creating a Custom Annotation to Handle Repetitive Logic

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LogExecutionTime {}

@Aspect
@Component
public class LoggingAspect {
    @Around("@annotation(LogExecutionTime)")
    public Object logTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed();
        log.info("{} took {}ms", pjp.getSignature(), System.currentTimeMillis() - start);
        return result;
    }
}
```
**What it does:** define the annotation (`@Retention(RUNTIME)` is critical), then write a Spring AOP `@Aspect` with `@Around` advice matching methods annotated with it. This is exactly how `@Transactional`, `@Cacheable`, and `@Async` work internally — annotation + AOP proxy intercepting the call.

> ⚠️ **Pitfall:** forgetting `@Retention(RetentionPolicy.RUNTIME)` on the custom annotation is the classic silent failure — without it, the AOP aspect's pointcut expression can't find the annotation via reflection at runtime, and the aspect simply never triggers, with no obvious error.

## Atomic Transactions and @Transactional

**What it does:** an atomic transaction means a group of operations either **all** succeed and commit together, or **all** fail and roll back together. Spring implements this declaratively via `@Transactional` — a proxy (JDK dynamic proxy for interface-based beans, CGLIB subclass proxy for concrete classes) wraps the method call, starting a transaction before the method runs and committing/rolling back after, based on whether the method completes normally or throws.

> ⚠️ **Pitfall:** by default, `@Transactional` rolls back only on **unchecked** exceptions (`RuntimeException`/`Error`) — checked exceptions do **not** trigger rollback unless you explicitly configure `rollbackFor = SomeCheckedException.class`. This is one of the most commonly missed/surprising details — a checked exception silently commits whatever happened before it unless `rollbackFor` is set.

## The Self-Invocation Gotcha

**What happens when an @Transactional method calls another @Transactional method:** by default (`Propagation.REQUIRED`), if called from **outside** any existing transaction, a new transaction starts; if called from within an existing transaction (one `@Transactional` method calling another on a *different* bean), the inner call **joins** the existing transaction — they share the same transaction, and a rollback in either affects both.

> ⚠️ **Pitfall — the critical gotcha:** if a method calls **another `@Transactional` method on the same class/bean** (self-invocation, `this.otherMethod()`), the transactional behavior is **completely bypassed** — because `@Transactional` works via AOP proxying, and calling a method on `this` directly skips the proxy entirely. Fix: either restructure (extract to a separate bean/service) or self-inject a proxy of the same bean.

## Private Methods, Final Methods/Classes — Silent @Transactional Failures

**Private methods:** no effect. `@Transactional`'s proxy-based mechanism (JDK dynamic proxy or CGLIB subclass) can only intercept calls that go **through the proxy**, and `private` methods aren't visible for either proxying strategy. Spring does **not** throw an error — the annotation is silently ignored.

**Final methods/classes:** `@Transactional` on a `final` method or class doesn't work either — Spring's proxy-based AOP uses CGLIB **subclass** proxies for concrete classes, and subclassing is exactly what `final` prevents.

> ⚠️ **Pitfall:** `@Transactional` has **three** distinct silent-failure modes — private methods, final methods/classes, and self-invocation via `this.` — all rooted in the same cause: something bypassing or blocking the AOP proxy.

## rollbackFor / noRollbackFor

`rollbackFor = SomeCheckedException.class` — expands rollback to include specific checked exceptions beyond the unchecked-only default. `noRollbackFor = SomeException.class` — the inverse: suppresses rollback for a specific exception type even if it would otherwise trigger one.

## Programmatic vs Declarative Transaction Management

```java
@Autowired private PlatformTransactionManager txManager;
public void doInTransaction() {
    new TransactionTemplate(txManager).execute(status -> {
        // business logic
        return null;
    });
}
```
**Declarative** (`@Transactional`) — the default, idiomatic approach; transaction boundaries defined by annotation, Spring's AOP proxy handles begin/commit/rollback automatically. **Programmatic** (`TransactionTemplate` or `PlatformTransactionManager` directly) — explicit, code-level control, useful when only part of a method needs to be transactional, or the boundary is conditional/dynamic at runtime.

> ⚠️ **Pitfall:** reach for `TransactionTemplate` specifically when method-level `@Transactional` would wrap *more* code in a transaction than you actually want.

## REQUIRES_NEW vs NESTED Propagation

**REQUIRES_NEW** — suspends any existing transaction and starts a completely independent new one; the outer resumes once the inner completes (commit or rollback), entirely independent of the outer's outcome. Use for things that must persist regardless of the outer transaction's fate (audit logs, notifications).

**NESTED** — runs *within* the same physical transaction but establishes a savepoint; if the nested portion rolls back, only work back to that savepoint is undone — the outer transaction remains active. Requires the DB/transaction manager to support savepoints — without it, `NESTED` silently degrades to `REQUIRED`.

**Rollback behavior:** a `REQUIRES_NEW` inner rollback never touches the outer transaction (physically separate). A `NESTED` inner rollback only reverts to its savepoint, but if the *outer* transaction later rolls back, everything (including the already-completed nested work) rolls back too.

> ⚠️ **Pitfall:** the savepoint-support fallback for `NESTED` (silently degrading to `REQUIRED` if unsupported) is a common gap — verify your specific database/transaction manager combination actually supports savepoints before relying on `NESTED`.

## Transactions Across Multiple Datasources

By default, a single `PlatformTransactionManager` is scoped to **one** datasource. Multiple datasources require configuring multiple transaction managers, and specifying which per method: `@Transactional("secondaryTxManager")`.

**Local vs distributed:** local transactions involve a single resource. Global/distributed transactions span multiple resources (two databases, or a database plus a message queue) coordinated via two-phase commit — requires JTA and a JTA-capable transaction manager (Atomikos, Bitronix).

> ⚠️ **Pitfall:** having `@Transactional` methods that each touch a different datasource does **not** give you atomicity *across* those datasources unless you've explicitly set up JTA — a common false assumption.

## Transaction Manager, Connection Pools, Manual Rollback, and Threads

```java
@Transactional
public void process() {
    if (someCondition) {
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
    }
}
```
The transaction manager obtains a connection from the pool at transaction start, manages commit/rollback against it, and returns it afterward. Manual rollback trigger (without throwing): `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`.

> ⚠️ **Pitfall:** transactions are **thread-bound** — if a `@Transactional` method spawns a new thread internally, that new thread does **not** automatically participate in the calling thread's transaction context. Starting a background thread from inside a `@Transactional` method and assuming its DB writes are covered by that transaction is a real, subtle bug.

## Common Transaction Pitfalls Checklist & Debugging

**Checklist:** self-invocation bypassing the proxy, private/final methods silently ignored, not specifying `rollbackFor` for checked exceptions, mixing multiple transaction managers without specifying which per method, applying `@Transactional` to a bean Spring doesn't actually manage.

**Debugging:** enable `logging.level.org.springframework.transaction=DEBUG` to see begin/commit/rollback boundaries, verify the proxy is actually being created, confirm which exception types actually propagated.

> ⚠️ **Pitfall:** a caught-and-swallowed exception inside the method never reaches the proxy to trigger rollback at all — that's not a transaction misconfiguration, the exception simply never reaches the transactional boundary in the first place.

---

## Interview Q&A

**Q: How would you create a custom annotation in Spring to handle repetitive logic?**
Covered above.

**Q: What happens when an @Transactional method calls another @Transactional method?**
Covered above.

**Q: Can @Transactional be applied to private methods, and does the compiler warn you?**
Covered above.

**Q: What do rollbackFor/noRollbackFor control, and why don't transactions work on final methods or classes?**
Covered above.

**Q: AOP fundamentals — Aspect, Advice, Pointcut, Join Point, Weaving, and writing a custom @Aspect.**
Covered above.

**Q: What are atomic transactions, and how are they implemented in Spring?**
Covered above.

**Q: Programmatic vs declarative transaction management, and when do you reach for TransactionTemplate?**
Covered above.

**Q: REQUIRES_NEW vs NESTED propagation — full comparison of behavior and rollback semantics.**
Covered above.

**Q: How does Spring handle transactions across multiple datasources, and can it do genuinely distributed transactions?**
Covered above.

**Q: How does the transaction manager interact with connection pools, how do you manually trigger a rollback, and how does @Transactional behave across threads?**
Covered above.

**Q: What's a practical checklist of common Spring transaction pitfalls, and how do you debug transaction issues?**
Covered above.
