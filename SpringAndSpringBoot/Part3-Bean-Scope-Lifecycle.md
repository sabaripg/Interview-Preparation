# ♻️ Part 3 — Bean Scope & Lifecycle

> Neat, point-based format with callout boxes, tables, and icons. Interview Q&A at the end.

---

## 🎭 Default Bean Scope — Singleton vs Prototype

| Scope | Behavior |
|---|---|
| **Singleton** (default) | Exactly one shared instance per container, created once (eagerly by default), reused everywhere |
| **Prototype** | A **new instance** every time it's requested/injected — container doesn't manage its full lifecycle after creation (no auto `@PreDestroy`) |

- Use **singleton** for stateless services (the overwhelming majority of beans).
- Use **prototype** when a bean genuinely needs mutable, request/call-specific state that shouldn't be shared.

> [!WARNING]
> Injecting a prototype into a singleton via normal field/constructor injection resolves the prototype **once** at singleton-creation time, then behaves like a singleton itself. Use `ObjectProvider<T>`, `@Lookup`, or a scoped proxy (Part 2) to actually get a fresh instance each time.

---

## 📏 The Full List of Scopes

- `singleton` (default) — one instance per container.
- `prototype` — new instance per request/injection.
- `request` — one instance per HTTP request (web-aware contexts only).
- `session` — one instance per HTTP session.
- `application` — one instance per `ServletContext`.
- `websocket` — one instance per WebSocket session.

> [!CAUTION]
> The web-specific scopes (`request`/`session`/`application`/`websocket`) only work in a web-aware `ApplicationContext` — using them in a non-web context throws an exception at bean-creation time.

---

## 🔄 The Full Ordered Bean Lifecycle

1. **Constructor** — object instantiated, constructor-injected dependencies available immediately.
2. **Property/setter injection** — any `@Autowired` setters/fields populated.
3. **Initialization callbacks**, in order: `@PostConstruct` → `InitializingBean.afterPropertiesSet()` → custom init method (`@Bean(initMethod = "init")`).
4. **Bean ready for use** — registered in the `ApplicationContext`.
5. **Destruction callbacks** (on shutdown), in order: `@PreDestroy` → `DisposableBean.destroy()` → custom destroy method (`@Bean(destroyMethod = "cleanup")`).

**Why three redundant hook mechanisms exist:**

| Mechanism | Why it exists |
|---|---|
| `@PostConstruct`/`@PreDestroy` | Standard `jakarta.annotation`, framework-agnostic — the modern default |
| `InitializingBean`/`DisposableBean` | Spring-specific interfaces, predate annotations, couple your class to Spring |
| Custom init/destroy via `@Bean` attributes | For classes you don't own the source of |

> [!IMPORTANT]
> Constructor-injected fields are available in the constructor itself; anything requiring setter/field-injected dependencies must wait for `@PostConstruct`. Calling a method that touches a setter-injected field directly inside the constructor is a real, subtle bug — that field is still `null` at that point.

---

## 🚀 @PostConstruct — a Real-World Use Case

- **Why not just the constructor:** the constructor runs **before** DI completes for field/setter-injected dependencies — anything depending on injected fields being non-null must wait until `@PostConstruct`, which the container guarantees runs after all injection finishes.
- (Constructor-injected fields are the exception — those ARE available in the constructor itself.)

**Real use cases:**
- Warming up/loading a cache from the database at startup.
- Validating required configuration properties are present/valid immediately (fail-fast).
- Initializing a connection to an external system once all dependencies are guaranteed available.

---

## 🐢 Eager vs Lazy Bean Creation

| Scope | Creation timing |
|---|---|
| **Singleton (default)** | Eagerly, at startup, during `ApplicationContext.refresh()` — this is why a missing dependency fails the app immediately (fail-fast) |
| **Prototype** | On every `getBean()`/injection request — never eager |
| **`@Lazy` singleton** | Deferred until first actual use — must be explicitly opted into |

```java
@Component
@Lazy
public class HeavyReportService {
    // loads a large dataset on creation — don't want this at startup if reports are rarely used
}
```

**When to use `@Lazy`:**
- The bean is expensive to create and rarely used.
- It depends on an external resource that may not be available at startup.
- It's a quick fix to break a circular dependency.

**When not to use it:**
- Most production singletons — you want fail-fast startup validation, especially for beans wrapping a resource that should be verified at startup (DB pool, Kafka connection).

> [!WARNING]
> `@Lazy` trades startup safety for runtime risk — a misconfigured lazy bean won't surface as a startup failure, it'll surface as a runtime exception on whatever request triggers its first use — a worse debugging experience in production.

---

## 🐛 Why @PostConstruct + @Transactional Silently Doesn't Work

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

- The AOP proxy implementing `@Transactional` is created in `BeanPostProcessor.postProcessAfterInitialization()` — which runs **after** `@PostConstruct`, not before.
- Any call from inside `@PostConstruct` — even to a method on the same bean — goes through the raw, not-yet-proxied object; transactional behavior isn't wired up yet.

**Fix** — implement `ApplicationListener<ContextRefreshedEvent>` (or listen for `ApplicationReadyEvent`) instead:

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

> [!CAUTION]
> No exception is thrown — this is a **silent correctness bug**, not a startup failure. The method just runs without a transaction, and the first sign of trouble is usually a partial/inconsistent write discovered much later.

---

## 📋 Interview Q&A

| Question | Short answer |
|---|---|
| Default bean scope vs Prototype? | Singleton = one shared instance; Prototype = new instance every request |
| All the scopes in Spring? | singleton, prototype, request, session, application, websocket |
| Real use case for @PostConstruct? | Cache warm-up, fail-fast config validation, external connection init |
| The full ordered bean lifecycle? | Constructor → setters → init callbacks → ready → destroy callbacks |
| 3 ways to inject a fresh prototype into a singleton? | ApplicationContext.getBean(), @Lookup, scoped proxy |
| Eager vs Lazy — when do you reach for @Lazy? | Expensive/rarely-used beans, or breaking circular deps — trades startup safety for runtime risk |
| Why does @Transactional inside @PostConstruct silently fail? | AOP proxy isn't created until after @PostConstruct runs |
