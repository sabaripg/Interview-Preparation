# Part 2 — Dependency Injection

> Injection styles, @Autowired vs @Inject, resolving ambiguity, circular dependencies. Interview Q&A at the end.

## @Autowired vs @Inject

**What they are:** `@Autowired` is Spring-specific; `@Inject` is the standard Java annotation (JSR-330, `javax.inject`/`jakarta.inject`), framework-agnostic.

**Why the difference matters:** functionally nearly identical for basic injection — the concrete difference is `@Autowired` has a `required` attribute (`@Autowired(required = false)`) allowing optional injection directly, while `@Inject` has no such attribute (you'd need `Optional<T>` or `@Nullable` instead).

**When to prefer `@Inject`:** when you want your code less tightly coupled to Spring specifically (portability across DI frameworks) — though in practice most Spring codebases just use `@Autowired` since they're already committed to Spring.

> ⚠️ **Pitfall:** the `required` attribute is the concrete functional difference to know — not just "one is Spring's, one is the standard."

## Constructor Injection — Why It's the Recommended Default

**What it is:** dependencies are passed in through the constructor, typically into `final` fields.

**Why it's preferred:** makes required dependencies explicit and immutable (`final` fields), fails fast at startup if a dependency is missing (rather than a later NPE), and makes the class trivially unit-testable without a Spring container — just `new ClassName(mock1, mock2)`.

```java
@Component
public class OrderService {
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

> ⚠️ **Pitfall — the "compile-time" myth:** never say "constructor injection fails at compile time if a dependency is missing." The Java compiler has zero knowledge of Spring's bean graph — it only validates Java syntax. Constructor injection is fail-fast at **runtime**, specifically at **application startup**, during `ApplicationContext.refresh()`:
```
Missing dependency → app refuses to start:
UnsatisfiedDependencyException: No qualifying bean of type 'PaymentService' available

vs. field injection with a missing dependency:
App STARTS SUCCESSFULLY.
NullPointerException thrown on the first request that touches the null field —
could be 3am in production before anyone notices.
```
Saying "fails at compile time" is one of the most common precision mistakes on this exact topic — always frame it as "fail-fast at startup."

### Why a final @Autowired field won't compile

```java
@Component
public class Server {
    @Autowired
    private final WebServer webServer; // COMPILE ERROR
}
```
Java requires `final` fields to be initialized at declaration or inside a constructor. Field injection happens via reflection *after* construction, so a `final` field annotated this way is never actually initialized by the time the constructor finishes — the compiler rejects it outright.

**Fix:** constructor injection — the field is genuinely set during construction, satisfying Java's `final` rules while also making the dependency required and immutable. This is a concrete illustration of *why* the constructor-injection recommendation isn't just style preference — it's the only injection style compatible with `final` fields at all.

## Setter Injection — When It's Actually the Right Choice

**What it is:** dependencies passed via setter methods after construction.

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

**When to use it:** genuinely **optional** dependencies where the bean should still function correctly without it, or when a dependency needs to be **reconfigured** after the bean is already constructed (plugin-style systems).

**When not to use it:** mandatory dependencies — the object can exist in a half-wired state before the setter actually runs, and fields can't be `final`.

> ⚠️ **Pitfall:** setter injection is sometimes reached for as a workaround for circular dependencies (Spring can inject an early, partially-built reference through a setter, but not through a constructor). That "works," but it's treating a symptom — the actual fix is almost always refactoring to remove the cycle.

## Field Injection — Why It's Discouraged in Production

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
| | Hides real dependency count — classes can silently balloon to 15+ fields |
| | Dependencies can be `null` if the Spring context hasn't loaded them yet |

**Why not to use it:** in production code (use constructor injection instead), whenever you need `final` fields, or whenever you plan to unit test without a Spring context — field-injected mocks require `ReflectionTestUtils` or similar rather than a plain `new OrderService(mockPayment)`.

## Both Constructor and Setter Injection on the Same Class

**What happens:** both are legal simultaneously — Spring first resolves and calls the constructor (injecting constructor-parameter dependencies), then applies setter injection for any `@Autowired`-annotated setters afterward, in the normal sequence (construct → populate properties/setters → `@PostConstruct`).

> ⚠️ **Pitfall:** this works, but flag it as poor practice regardless — mixing styles obscures which dependencies are truly required (constructor) vs optional (setter), undermining the clarity constructor injection is meant to provide.

## @Primary vs @Qualifier

**What they do:** `@Primary` marks one bean among multiple candidates of the same type as the **default** to inject when no further disambiguation is specified — a blanket, type-level preference. `@Qualifier("beanName")` explicitly specifies **which** bean to inject at a specific injection point by name, overriding whatever `@Primary` might otherwise select.

**When to use which:** `@Primary` when one implementation is genuinely the sensible default almost everywhere. `@Qualifier` for specific injection points that need a non-default implementation, or when there's no clear "default" and every site should be explicit.

> ⚠️ **Pitfall:** `@Qualifier` at an injection site always wins over `@Primary` — know this precedence order explicitly, it's a common precise-detail check.

## Resolving Ambiguity Without @Qualifier

- Use `@Primary` on the preferred implementation, if there's a sensible default.
- Name the field/parameter to exactly match the target bean's name — Spring falls back to matching by **bean name** when there are multiple candidates and the injection point's variable name matches one exactly.
- Use a more specific type — injecting the concrete class directly instead of a shared interface removes the ambiguity (though it reduces the benefit of coding to an interface).
- Restructure to avoid needing multiple implementations of the same interface injected together in the first place — sometimes a sign a factory or strategy map would be cleaner.

> ⚠️ **Pitfall:** the variable-name-matching fallback is a genuinely lesser-known mechanism — mentioning it alongside `@Primary` shows deeper familiarity with Spring's resolution algorithm.

## How Spring Handles Circular Dependencies

**Setter/field injection:** Spring can resolve circular dependencies by creating early, not-fully-initialized bean references and injecting them, then completing initialization afterward — works because setter injection doesn't require the dependency to be fully ready at construction time.

**Constructor injection:** Spring **cannot** resolve true circular dependencies — both beans need a fully-constructed instance of the other at construction time, which is impossible. The application fails to start with `BeanCurrentlyInCreationException`.

> ⚠️ **Pitfall:** Spring Boot 2.6+ disables circular-reference resolution **by default**, even for setter injection, specifically to surface these as explicit startup failures rather than silently allowing the pattern. `spring.main.allow-circular-references=true` re-enables the old lenient behavior if genuinely needed — but the real fix is usually refactoring to remove the circular dependency, not enabling the workaround flag. Mentioning this default change signals up-to-date knowledge.

## Injecting a Prototype Bean into a Singleton — Getting a Fresh Instance Each Time

**The trap:** injecting a prototype bean into a singleton via normal field/constructor injection resolves the prototype **once**, at singleton-creation time — it then effectively behaves like a singleton itself.

**Three real fixes:**
1. **Inject `ApplicationContext` directly**, call `context.getBean(PrototypeBean.class)` each time you need a fresh instance — simplest, but couples the class to the Spring container API.
2. **`@Lookup` method injection** — declare an abstract (or any-body) method returning the prototype type; Spring overrides it at runtime via a CGLIB-generated subclass to return a fresh instance on every call.
3. **Scoped proxy** (`@Scope(value="prototype", proxyMode=ScopedProxyMode.TARGET_CLASS)`) — Spring injects a proxy that transparently fetches a new prototype instance on each method call through it.

---

## Interview Q&A

**Q: Difference between @Autowired and @Inject?**
Covered above.

**Q: When would you choose setter injection over constructor injection, and vice versa?**
Covered above — constructor injection is the current recommended default; setter reserved for optional dependencies or last-resort circular-dependency workarounds.

**Q: Can we avoid dependency ambiguity without using @Qualifier?**
Covered above.

**Q: Why won't a class with a final @Autowired field compile, and how does constructor injection fix it?**
Covered above.

**Q: Difference between @Primary and @Qualifier?**
Covered above.

**Q: How does Spring handle circular dependencies?**
Covered above.

**Q: What happens if both setter-based and constructor-based injection are applied to the same class?**
Covered above.

**Q: What are the three ways to inject a prototype-scoped bean into a singleton so you actually get a fresh instance each time?**
Covered above.

**Q: Field Injection — full pros/cons, and why it's discouraged in production code.**
Covered above.

**Q: Setter Injection — when is it actually the right choice?**
Covered above.

**Q: Constructor Injection — the "Compile-Time" myth, and why it's still the recommended default.**
Covered above.
