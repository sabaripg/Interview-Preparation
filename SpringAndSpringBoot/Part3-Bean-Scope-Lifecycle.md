# Part 3 — Bean Scope & Lifecycle

> Singleton vs prototype and the other scopes, the full bean lifecycle, eager vs lazy creation, and a genuinely tricky @PostConstruct + @Transactional gotcha. Interview Q&A at the end.

## Default Bean Scope — Singleton vs Prototype

**What they are:** the default scope, **singleton**, means exactly one shared instance per Spring container, created once (eagerly, by default) and reused for every injection point. **Prototype** means a **new instance** is created every time the bean is requested/injected — the container doesn't manage its full lifecycle after creation (no automatic `@PreDestroy` callback, for instance).

**When to use which:** singleton for stateless services (the overwhelming majority of Spring beans). Prototype when a bean genuinely needs to hold mutable, request/call-specific state that shouldn't be shared.

> ⚠️ **Pitfall:** injecting a prototype bean into a singleton via normal field/constructor injection is a classic trap — the prototype gets resolved **once** at singleton-creation time and then effectively behaves like a singleton. You need `ObjectProvider<T>`, `@Lookup`, or a scoped proxy to actually get a fresh instance each time (see Part 2).

## The Full List of Scopes

- `singleton` (default) — one instance per container.
- `prototype` — new instance per request/injection.
- `request` — one instance per HTTP request (web-aware contexts only).
- `session` — one instance per HTTP session.
- `application` — one instance per `ServletContext`.
- `websocket` — one instance per WebSocket session.

> ⚠️ **Pitfall:** the web-specific scopes (`request`/`session`/`application`/`websocket`) only work in a web-aware `ApplicationContext` — using them in a non-web context throws an exception at bean-creation time.

## The Full Ordered Bean Lifecycle

**What it is, step by step:**
1. **Constructor** — object instantiated, constructor-injected dependencies available immediately.
2. **Property/setter injection** — any `@Autowired` setters/fields populated.
3. **Initialization callbacks**, in order if multiple are present: `@PostConstruct`-annotated method → `InitializingBean.afterPropertiesSet()` → custom init method (`@Bean(initMethod = "init")`).
4. **Bean ready for use** — registered in the `ApplicationContext`, available for injection elsewhere.
5. **Destruction callbacks** (on container shutdown), in order: `@PreDestroy`-annotated method → `DisposableBean.destroy()` → custom destroy method (`@Bean(destroyMethod = "cleanup")`).

**Why three redundant hooks exist:** for historical/compatibility reasons — `@PostConstruct`/`@PreDestroy` (standard `jakarta.annotation`, framework-agnostic) is the modern default; `InitializingBean`/`DisposableBean` (Spring-specific interfaces) predate annotations and couple your class directly to Spring; custom init/destroy methods via `@Bean` attributes are for classes you don't own the source of.

> ⚠️ **Pitfall:** constructor-injected fields are available in the constructor itself; anything requiring setter/field-injected dependencies must wait for `@PostConstruct`. Calling a method that touches a setter-injected field directly inside the constructor is a real, subtle bug since that field is still `null` at that point.

## @PostConstruct — a Real-World Use Case

**Why you need it instead of the constructor:** the constructor runs **before** dependency injection completes for field/setter-injected dependencies, so any logic depending on injected fields being non-null must wait until `@PostConstruct`, which the container guarantees runs after all injection is finished. (Constructor-injected fields are the exception — those ARE available in the constructor itself.)

**Real use cases:** warming up/loading a cache from the database at startup, validating that required configuration properties are present/valid immediately (fail-fast), initializing a connection to an external system once all dependencies are guaranteed available.

## Eager vs Lazy Bean Creation

**Singleton (default)** — created eagerly at startup, during `ApplicationContext.refresh()`. This is why a missing dependency or bad config fails the app immediately at startup rather than on the first request in production (fail-fast).

**Prototype** — created on every `getBean()`/injection request, never eager.

**`@Lazy` singleton** — deferred until first actual use. Must be explicitly opted into.

```java
@Component
@Lazy
public class HeavyReportService {
    // loads a large dataset on creation — don't want this at startup if reports are rarely used
}
```

**When to use `@Lazy`:** the bean is expensive to create and rarely used, it depends on an external resource that may not be available at startup, or it's a quick fix to break a circular dependency.

**When not to use it:** most production singletons — you want fail-fast startup validation, especially for beans wrapping a resource that should be verified at startup (DB pool, Kafka connection).

> ⚠️ **Pitfall:** `@Lazy` trades startup safety for runtime risk — a misconfigured lazy bean won't surface as a startup failure, it'll surface as a runtime exception on whatever request happens to trigger its first use, a worse debugging experience in production.

## Why Calling a @Transactional Method from @PostConstruct Silently Doesn't Work

```java
@Service
public class UserService {
    @PostConstruct
    public void init() {
        loadDefaultUsers(); // calls the @Transactional method below
    }

    @Transactional
    public void loadDefaultUsers() {
        // NO transaction here — proxy doesn't exist yet at @PostConstruct time
        userRepo.save(new User("admin"));
    }
}
```

**Why it fails:** the AOP proxy that implements `@Transactional` (and `@Async`, `@Cacheable`) is created in `BeanPostProcessor.postProcessAfterInitialization()` — which runs **after** `@PostConstruct`, not before. So any call made from inside `@PostConstruct` — even to a method on the same bean — goes through the raw, not-yet-proxied object; the transactional behavior simply isn't wired up yet.

**Fix:** implement `ApplicationListener<ContextRefreshedEvent>` (or listen for `ApplicationReadyEvent`) instead of doing this work in `@PostConstruct` — by the time that event fires, the full context including all proxies is ready.

```java
@Service
public class UserService implements ApplicationListener<ContextRefreshedEvent> {
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        loadDefaultUsers(); // proxy exists now
    }

    @Transactional
    public void loadDefaultUsers() {
        userRepo.save(new User("admin")); // transaction active
    }
}
```

> ⚠️ **Pitfall:** no exception is thrown — this is a silent correctness bug, not a startup failure. The method just runs without a transaction, and the first sign of trouble is usually a partial/inconsistent write discovered much later.

---

## Interview Q&A

**Q: What's the default bean scope, and how does it differ from Prototype?**
Covered above.

**Q: What are the different scopes in Spring?**
Covered above.

**Q: Example of a real-world use case where @PostConstruct is particularly useful?**
Covered above.

**Q: What happens if both setter-based and constructor-based injection are applied to the same class?**
See Part 2.

**Q: What's the full ordered bean lifecycle, and what are the alternative hook points at each stage?**
Covered above.

**Q: What are the three ways to inject a prototype-scoped bean into a singleton so you actually get a fresh instance each time?**
See Part 2.

**Q: Eager vs Lazy bean creation — what's the difference, and when do you actually reach for @Lazy?**
Covered above.

**Q: Why does calling a @Transactional method from inside @PostConstruct silently not work?**
Covered above.
