# 🔀 Part 5 — AOP, Proxies & Transactions

> Neat, point-based format with callout boxes, tables, and icons. Interview Q&A at the end.

---

## 🕸️ AOP Fundamentals — Aspect, Advice, Pointcut, Join Point, Weaving

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

> [!CAUTION]
> `@Around` fully replaces the method's execution flow — **forgetting `pjp.proceed()` means the real method silently never runs at all**, a no-op bug with no exception thrown. This underlying proxy mechanism is exactly why `final` classes/methods and `private` methods silently break `@Transactional`/`@Async`/`@Cacheable` too — CGLIB needs to subclass and override, which `final`/`private` block.

---

## ✏️ Creating a Custom Annotation to Handle Repetitive Logic

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
- Define the annotation (`@Retention(RUNTIME)` is critical), then write an `@Aspect` with `@Around` advice matching it.
- This is exactly how `@Transactional`, `@Cacheable`, and `@Async` work internally.

> [!WARNING]
> Forgetting `@Retention(RetentionPolicy.RUNTIME)` is the classic silent failure — without it, the aspect's pointcut can't find the annotation via reflection at runtime, and the aspect simply never triggers, no error.

---

## ⚛️ Atomic Transactions and @Transactional

- **Atomic** = a group of operations either **all** succeed and commit together, or **all** fail and roll back together.
- Implemented via a proxy (JDK dynamic proxy or CGLIB) wrapping the method — starts a transaction before, commits/rolls back after, based on whether the method completes or throws.

> [!IMPORTANT]
> By default, `@Transactional` rolls back only on **unchecked** exceptions (`RuntimeException`/`Error`). Checked exceptions do **not** trigger rollback unless you explicitly set `rollbackFor = SomeCheckedException.class`. One of the most commonly missed details.

---

## 🪞 The Self-Invocation Gotcha

| Scenario | What happens |
|---|---|
| `@Transactional` method called from **outside** any transaction | New transaction starts |
| Called from within an existing transaction (different bean) | Joins the existing transaction — shared rollback |
| **Self-invocation** (`this.otherMethod()` on the same bean) | ❌ Transactional behavior **completely bypassed** — the call skips the proxy entirely |

> [!CAUTION]
> The self-invocation bypass is a critical, easy-to-miss gotcha. Fix: restructure (extract to a separate bean/service) or self-inject a proxy of the same bean.

---

## 🔇 Private Methods, Final Methods/Classes — Silent @Transactional Failures

| Case | Why it fails |
|---|---|
| **Private methods** | Proxies can't intercept calls that don't go through them — `private` isn't visible to either proxying strategy. Spring throws **no error**. |
| **Final methods/classes** | CGLIB needs to **subclass** to proxy — `final` prevents subclassing entirely |

> [!CAUTION]
> `@Transactional` has **three** distinct silent-failure modes — private methods, final methods/classes, and self-invocation — all rooted in the same cause: something bypassing or blocking the AOP proxy.

---

## ↩️ rollbackFor / noRollbackFor

- `rollbackFor = SomeCheckedException.class` — expands rollback to specific checked exceptions.
- `noRollbackFor = SomeException.class` — suppresses rollback for a specific type even if it'd otherwise trigger one.

---

## 🎛️ Programmatic vs Declarative Transaction Management

```java
@Autowired private PlatformTransactionManager txManager;
public void doInTransaction() {
    new TransactionTemplate(txManager).execute(status -> {
        // business logic
        return null;
    });
}
```

| Approach | When to use |
|---|---|
| **Declarative** (`@Transactional`) | Default, idiomatic — AOP proxy handles begin/commit/rollback automatically |
| **Programmatic** (`TransactionTemplate`) | When only part of a method needs to be transactional, or the boundary is conditional/dynamic |

---

## 🪆 REQUIRES_NEW vs NESTED Propagation

| | REQUIRES_NEW | NESTED |
|---|---|---|
| Behavior | Suspends existing transaction, starts a fully independent new one | Runs within the same physical transaction, using a savepoint |
| Rollback scope | Inner rollback never touches the outer — physically separate | Inner rollback only reverts to its savepoint; outer rollback undoes everything including nested work |
| Requires | Nothing special | DB/transaction manager savepoint support |
| Use case | Audit logs, notifications that must persist regardless of outer outcome | Partial rollback within one larger unit of work |

> [!WARNING]
> Without savepoint support, `NESTED` silently degrades to `REQUIRED` — verify your DB/transaction manager combination actually supports savepoints before relying on it.

---

## 🗄️ Transactions Across Multiple Datasources

- By default, one `PlatformTransactionManager` is scoped to **one** datasource.
- Multiple datasources → multiple transaction managers, specified per method: `@Transactional("secondaryTxManager")`.
- **Local** transactions = single resource. **Distributed/global** transactions span multiple resources via two-phase commit — needs JTA (Atomikos, Bitronix).

> [!CAUTION]
> Having `@Transactional` methods each touching a different datasource does **not** give atomicity *across* them unless you've set up JTA explicitly.

---

## 🧵 Transaction Manager, Connection Pools, Manual Rollback, and Threads

```java
@Transactional
public void process() {
    if (someCondition) {
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
    }
}
```
- The transaction manager obtains a pooled connection at transaction start, manages commit/rollback, returns it afterward.
- Manual rollback (without throwing): `setRollbackOnly()`.

> [!WARNING]
> Transactions are **thread-bound** — a `@Transactional` method spawning a new thread does **not** carry its transaction context to that thread. DB writes on the new thread are outside the original transaction's boundary.

---

## ✅ Common Transaction Pitfalls Checklist & Debugging

- Self-invocation bypassing the proxy.
- Private/final methods silently ignored.
- Missing `rollbackFor` for checked exceptions that need rollback.
- Multiple transaction managers without specifying which per method.
- `@Transactional` on a bean Spring doesn't actually manage.

**Debugging:** `logging.level.org.springframework.transaction=DEBUG`, verify the proxy is created, confirm which exceptions actually propagated.

> [!CAUTION]
> A caught-and-swallowed exception inside the method **never reaches the proxy** to trigger rollback — that's not a misconfiguration, the exception simply never reaches the transactional boundary.

---

## 📋 Interview Q&A

| Question | Short answer |
|---|---|
| Custom annotation for repetitive logic? | @interface + @Retention(RUNTIME) + @Aspect with @Around |
| @Transactional calling another @Transactional method? | Joins if from another bean; bypassed entirely on self-invocation |
| @Transactional on private methods? | Silently ignored — no error, no transaction |
| rollbackFor/noRollbackFor, and final methods/classes? | Expand/suppress rollback exceptions; final blocks CGLIB subclassing |
| AOP fundamentals — Aspect/Advice/Pointcut/Join Point/Weaving? | See table above |
| What are atomic transactions? | All-or-nothing group of operations via AOP proxy |
| Programmatic vs declarative transactions? | TransactionTemplate for partial/conditional boundaries |
| REQUIRES_NEW vs NESTED? | Independent transaction vs savepoint within the same one |
| Transactions across multiple datasources? | Multiple tx managers; true atomicity needs JTA |
| Transaction manager + connection pools + threads? | Pool-scoped connections; transactions don't cross threads |
| Common transaction pitfalls checklist? | See list above |
