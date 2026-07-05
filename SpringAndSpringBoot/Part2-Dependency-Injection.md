# 💉 Part 2 — Dependency Injection

> Neat, point-based format with callout boxes, tables, and icons. Interview Q&A at the end.

---

## 🆚 @Autowired vs @Inject

- `@Autowired` — Spring-specific.
- `@Inject` — standard Java (JSR-330, `javax.inject`/`jakarta.inject`), framework-agnostic.
- Functionally nearly identical for basic injection.

| Feature | `@Autowired` | `@Inject` |
|---|---|---|
| `required` attribute | ✅ `@Autowired(required = false)` | ❌ none — use `Optional<T>`/`@Nullable` instead |
| Framework coupling | Spring-specific | Portable across DI frameworks |

> [!IMPORTANT]
> The `required` attribute is the concrete functional difference to know — not just "one is Spring's, one is the standard."

---

## 🏗️ Constructor Injection — Why It's the Recommended Default

- Dependencies passed through the constructor, typically into `final` fields.
- Makes required dependencies **explicit and immutable**.
- **Fails fast at startup** if a dependency is missing, rather than a later NPE.
- Trivially unit-testable without a Spring container — just `new ClassName(mock1, mock2)`.

```java
@Component
public class OrderService {
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

> [!CAUTION]
> **The "compile-time" myth** — never say constructor injection "fails at compile time" if a dependency is missing. The compiler has zero knowledge of Spring's bean graph. It's fail-fast at **runtime**, specifically at **application startup** (`ApplicationContext.refresh()`):
> ```
> Missing dependency → app refuses to start:
> UnsatisfiedDependencyException: No qualifying bean of type 'PaymentService' available
>
> vs. field injection with a missing dependency:
> App STARTS SUCCESSFULLY.
> NullPointerException thrown on the first request that touches the null field —
> could be 3am in production before anyone notices.
> ```
> Always frame it as "fail-fast at startup," not compile time — this is one of the most common precision mistakes on this exact topic.

### Why a `final @Autowired` Field Won't Compile

```java
@Component
public class Server {
    @Autowired
    private final WebServer webServer; // COMPILE ERROR
}
```
- Java requires `final` fields to be initialized at declaration or inside a constructor.
- Field injection happens via reflection **after** construction — so a `final` field is never actually initialized by the time the constructor finishes, and the compiler rejects it.
- **Fix:** constructor injection — the field is genuinely set during construction, satisfying `final` while making the dependency required and immutable.

---

## 🔧 Setter Injection — When It's Actually the Right Choice

```java
@Component
public class ReportService {
    private MetricsService metricsService; // optional — report works without metrics

    @Autowired(required = false) // don't fail startup if no MetricsService bean exists
    public void setMetricsService(MetricsService metricsService) {
        this.metricsService = metricsService;
    }

    public void generateReport() {
        if (metricsService != null) {
            metricsService.record("report.generated");
        }
    }
}
```

**When to use it:**
- Genuinely **optional** dependencies — the bean should still function without it.
- A dependency needs to be **reconfigured** after construction (plugin-style systems).

**When not to use it:**
- Mandatory dependencies — the object can exist in a half-wired state before the setter runs, and fields can't be `final`.

> [!WARNING]
> Setter injection is sometimes used as a workaround for circular dependencies. That "works," but it's treating a symptom — the actual fix is almost always refactoring to remove the cycle.

---

## 🚫 Field Injection — Why It's Discouraged in Production

```java
@Component
public class OrderService {
    @Autowired
    private PaymentService paymentService; // Spring sets this via reflection AFTER construction
}
```

| Pros | Cons |
|---|---|
| Minimal boilerplate | Can't use `final` fields — no immutability |
| Easy to add a new dependency | Hard to unit test without Spring/reflection |
| | Hides real dependency count — classes silently balloon to 15+ fields |
| | Dependencies can be `null` if the context hasn't loaded them yet |

> [!WARNING]
> Avoid it in production code, whenever you need `final` fields, or whenever you plan to unit test without a Spring context — field-injected mocks require `ReflectionTestUtils` rather than plain `new OrderService(mockPayment)`.

---

## 🔀 Both Constructor and Setter Injection on the Same Class

- Both are legal simultaneously.
- Spring first calls the constructor (injecting constructor-parameter dependencies), then applies setter injection afterward — construct → populate setters → `@PostConstruct`.

> [!CAUTION]
> This works, but is poor practice — mixing styles obscures which dependencies are truly required (constructor) vs optional (setter).

---

## 🎯 @Primary vs @Qualifier

- `@Primary` — marks one bean among multiple candidates as the **default** to inject — a blanket, type-level preference.
- `@Qualifier("beanName")` — explicitly specifies **which** bean to inject at a specific injection point, by name — overrides `@Primary`.

| When to use | Annotation |
|---|---|
| One implementation is the sensible default almost everywhere | `@Primary` |
| A specific injection point needs a non-default implementation | `@Qualifier` |

> [!IMPORTANT]
> `@Qualifier` at an injection site **always wins** over `@Primary` — know this precedence order, it's a common precise-detail check.

---

## 🧭 Resolving Ambiguity Without @Qualifier

- Use `@Primary` on the preferred implementation, if there's a sensible default.
- Name the field/parameter to exactly match the target bean's name — Spring falls back to matching by **bean name** when the variable name matches one candidate exactly.
- Use a more specific type — injecting the concrete class directly instead of a shared interface removes ambiguity (reduces the benefit of coding to an interface, though).
- Restructure to avoid needing multiple implementations injected together in the first place.

> [!TIP]
> The variable-name-matching fallback is a lesser-known mechanism — mentioning it alongside `@Primary` shows deeper familiarity with Spring's resolution algorithm.

---

## 🔄 How Spring Handles Circular Dependencies

| Injection type | Can Spring resolve a cycle? |
|---|---|
| Setter/field injection | ✅ Yes — creates early, not-fully-initialized references, completes initialization afterward |
| Constructor injection | ❌ No — both beans need a fully-constructed instance of the other at construction time, which is impossible → `BeanCurrentlyInCreationException` |

> [!WARNING]
> **Spring Boot 2.6+ disables circular-reference resolution by default**, even for setter injection, to surface these as explicit startup failures. `spring.main.allow-circular-references=true` re-enables the old lenient behavior — but the real fix is almost always refactoring to remove the cycle. Mentioning this default change signals up-to-date knowledge.

---

## 🌀 Injecting a Prototype Bean into a Singleton — Getting a Fresh Instance Each Time

- **The trap:** injecting a prototype bean into a singleton via normal field/constructor injection resolves the prototype **once**, at singleton-creation time — it then behaves like a singleton itself.

**Three real fixes:**
1. **Inject `ApplicationContext` directly**, call `context.getBean(PrototypeBean.class)` each time — simplest, but couples the class to the container API.
2. **`@Lookup` method injection** — Spring overrides an abstract method via a CGLIB subclass to return a fresh instance on every call.
3. **Scoped proxy** (`@Scope(value="prototype", proxyMode=ScopedProxyMode.TARGET_CLASS)`) — the singleton holds a proxy that transparently fetches a new instance on each call through it.

---

## 📋 Interview Q&A

| Question | Short answer |
|---|---|
| @Autowired vs @Inject? | Same core mechanism; `@Autowired` has `required`, `@Inject` doesn't |
| Setter vs constructor injection — when to choose which? | Constructor is the default; setter for optional deps or circular-dep last resort |
| Avoiding ambiguity without @Qualifier? | `@Primary`, variable-name matching, more specific type, or restructure |
| Why won't a final @Autowired field compile? | Field injection sets it via reflection after construction — too late for `final` |
| @Primary vs @Qualifier? | `@Primary` = default; `@Qualifier` at injection site always wins |
| How does Spring handle circular dependencies? | Works for setter injection (pre-2.6 default), never for constructor injection |
| Mixing constructor + setter injection on one class? | Legal, but poor practice — obscures required vs optional |
| Getting a fresh prototype bean into a singleton? | `ApplicationContext.getBean()`, `@Lookup`, or a scoped proxy |
| Field injection — pros/cons? | Minimal boilerplate vs no immutability, poor testability, hidden dependency count |
| Setter injection — when is it right? | Genuinely optional dependencies, or reconfigurable ones |
| The "compile-time" myth for constructor injection? | It's fail-fast at startup (runtime), never compile time |
