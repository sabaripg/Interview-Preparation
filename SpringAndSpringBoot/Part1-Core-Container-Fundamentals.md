# 🌱 Part 1 — Core Container Fundamentals

> Neat, point-based format with callout boxes, tables, and icons. Interview Q&A at the end.

---

## 🧩 Why Spring Exists in the First Place

- Without a framework, every class that needs a collaborator has to build it directly — e.g. a service constructing `new SomeRepository(new SomeConnectionPool(...))` inside itself.
- This **tightly wires** the class to one specific way of building its dependencies, so it can't easily swap in a fake/mock version for testing without changing its own source code.
- Multiply this across a hundred classes and nothing can be tested independently, and changing how one low-level piece is built means hunting down every place that constructs it directly.
- **Dependency Injection (DI) / Inversion of Control (IoC)** solves this: the object no longer builds its own dependencies — a **container** builds them and hands them over, ready-made.
- The object just declares "I need a `PaymentService`" and trusts it'll be supplied — without knowing or caring who built it or how.
- This single inversion is what makes classes independently testable, and it's the foundation everything else in Spring builds on.

**Beyond DI, Spring also gives you:**
- Declarative handling of cross-cutting concerns (transactions, security, logging) via annotations like `@Transactional`/`@Secured` instead of hand-written plumbing.
- Thinner abstractions over verbose lower-level APIs — `JdbcTemplate`, `RestTemplate` instead of raw JDBC/HTTP client code.
- Spring Boot's **opinionated defaults**, cutting setup time from days of XML config to one annotated class + a `main()` method.

> [!TIP]
> If asked **"why Spring"** in an interview, don't answer "it's a framework for building Java apps" — lead with *why dependency injection matters* (testability, decoupling), since that's the root idea everything else hangs off of.

---

## 🫘 What a "Bean" Actually Is

- A **bean** is any object whose lifecycle (creation, wiring, destruction) is managed by the Spring container — not by application code calling `new`.
- The defining trait is **who manages it**, not what class it is — the exact same class is "just an object" in a plain unit test, and a "bean" only once Spring is responsible for it.

**Two ways to register a bean:**

| Approach | How it works |
|---|---|
| **Component scanning** | Annotate the class itself (`@Component` or a more specific cousin) — Spring finds and registers it automatically at startup |
| **Explicit `@Bean` method** | Write a method inside a `@Configuration` class — needed when you don't own the class's source (e.g. third-party library) |

- Beans are **singleton** by default — one shared instance for the whole application (other lifetimes exist too — see Part 3).
- You request an instance via `@Autowired` or, better, a constructor parameter — Spring supplies the already-built instance, you never call `new` yourself.

---

## 🏬 Where All of This Lives — the ApplicationContext

- The **`ApplicationContext`** is the "warehouse" holding every bean definition, and for singletons, the actual constructed instances too.
- Underneath it sits `BeanFactory` — the bare-bones registry that actually instantiates and wires beans.
- `ApplicationContext` extends `BeanFactory`, adding what almost every real app wants: event publishing, i18n messages, environment/profile awareness.

| | `BeanFactory` | `ApplicationContext` |
|---|---|---|
| Singleton creation | Lazy — only when first requested | **Eager** — at startup |
| Extra features | None — bare registry | Events, i18n, environment/profiles |
| Used in practice | Rarely constructed directly | Virtually always what Spring Boot uses |

> [!NOTE]
> "Container" and "ApplicationContext" get used interchangeably day to day — that's fine. A sharper answer distinguishes the *abstract idea* of a container from the *concrete interface* (`ApplicationContext`).

---

## 🏷️ The Stereotype Annotations — Four Names, Mostly One Mechanism

- `@Component`, `@Service`, `@Repository`, `@Controller` all mark a class as a Spring-managed bean.
- **Three of the four are just `@Component` wearing a costume** — `@Service`, `@Repository`, and `@Controller` are each themselves meta-annotated with `@Component`, which is why component scanning picks all of them up identically.

| Annotation | Layer | Extra behavior beyond `@Component`? |
|---|---|---|
| `@Component` | Anything | None — the generic form |
| `@Service` | Business logic | None — purely a naming convention |
| `@Controller` | Web (Spring MVC) | Returns view names by default (`@RestController` = `@Controller` + `@ResponseBody` on every method) |
| `@Repository` | Data access | ✅ **Exception translation** — native JDBC/JPA exceptions wrapped into Spring's unified `DataAccessException` hierarchy |

> [!IMPORTANT]
> If asked "what's the difference between these four," don't answer with vague vibes. Name the **one concrete functional difference** — `@Repository`'s exception translation — and be upfront the rest is organizational, not enforced differently.

---

## ⚙️ @Configuration vs @SpringBootConfiguration

- `@Configuration` — what you actually write; marks a class as a source of `@Bean` definitions (the modern replacement for XML config).
- `@SpringBootConfiguration` — a Spring Boot-specific annotation almost nobody applies by hand, bundled invisibly inside `@SpringBootApplication`.
- Its purpose: giving testing frameworks one unambiguous way to find "the one true primary configuration class" among potentially many `@Configuration` classes.

---

## 🧱 @Component vs @Bean — Whose Source Code Are You Looking At?

- **Your own class**, with a simple injectable constructor → use `@Component` on the class, picked up by scanning.
- **A third-party class** you can't annotate, or one needing **real construction logic** (conditional setup, multiple differently-configured instances) → use a `@Bean` method inside your own `@Configuration` class.

---

## 🔧 Registering Beans Programmatically (Advanced, Rare)

- Occasionally you need to register a bean definition *during* startup — e.g. multi-tenant systems needing per-tenant beans, or plugin architectures discovering modules at runtime.
- Solution: implement `BeanDefinitionRegistryPostProcessor`.

```java
@Component
public class DynamicBeanRegistrar implements BeanDefinitionRegistryPostProcessor {
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        registry.registerBeanDefinition("dynamicBean",
            BeanDefinitionBuilder.genericBeanDefinition(MyBean.class).getBeanDefinition());
    }
}
```

> [!WARNING]
> Be honest this is genuinely rare — framework-authoring/plugin territory, not day-to-day service work. Bringing it up unprompted as everyday practice undersells your judgment.

---

## 🏛️ Spring's Own Internals Are Built on Patterns You Already Know

| Pattern | Where it shows up in Spring |
|---|---|
| **Singleton** | Singleton-scoped beans |
| **Factory** | `BeanFactory`, `@Bean` factory methods |
| **Proxy** | The machinery behind `@Transactional`, `@Async`, `@Cacheable` (JDK dynamic proxy / CGLIB) |
| **Template Method** | `JdbcTemplate`, `RestTemplate`, `RabbitTemplate` — fixed skeleton, customizable middle step |
| **Front Controller** | `DispatcherServlet` — the single entry point every HTTP request passes through |

> [!TIP]
> Naming `DispatcherServlet` as "Front Controller" or the transactional proxy machinery as "Proxy pattern" signals you understand Spring as an *application* of general design principles — not framework-specific trivia memorized in isolation.

---

## 🔗 Parent-Child Contexts and Spring's Event System

- `ApplicationContext`s can be arranged **hierarchically** — a parent's beans are visible to a child context, never the reverse.
- Pre-Boot pattern: a root context holding services/repositories, a child `WebApplicationContext` holding just controllers. Spring Boot usually collapses this into one flat context today, but it still explains some `@ComponentScan` placement oddities in older codebases.
- Spring's **event system** — `ApplicationEventPublisher.publishEvent()` + `@EventListener` — lets beans talk without being directly wired together.

```java
@Service
public class OrderService {
    @Autowired
    private ApplicationEventPublisher publisher;

    public void placeOrder(Order order) {
        publisher.publishEvent(new OrderPlacedEvent(this, order));
    }
}

@Component
public class EmailNotifier {
    @EventListener
    public void onOrderPlaced(OrderPlacedEvent event) {
        sendConfirmationEmail(event.getOrder());
    }
}
```
- `OrderService` has no idea `EmailNotifier` exists — that's the entire point.

> [!CAUTION]
> Events are **synchronous by default** — `publishEvent()` blocks until every listener finishes, and an exception in any listener propagates back to the caller. Mark a listener `@Async` (with `@EnableAsync` enabled) if it shouldn't slow down or break the original operation — don't assume events are fire-and-forget by default.

---

## 📋 Interview Q&A

| Question | Short answer |
|---|---|
| Why do we need Spring? | Testability + decoupling via DI first, framework conveniences second |
| What exactly is a Spring Bean? | An object whose lifecycle the container manages — defined by *who's in charge*, not the class |
| Where are all beans stored? | The `ApplicationContext`, built on `BeanFactory` |
| What is a Spring Container? | The abstract IoC concept, concretely realized as `ApplicationContext` |
| @Component vs @Repository vs @Service vs @Controller? | All beans; `@Repository`'s exception translation is the one real functional difference |
| @Configuration vs @SpringBootConfiguration? | You write the former; the latter is a hidden test-discovery marker in `@SpringBootApplication` |
| @Component vs @Bean? | Do you own the class's source, and does construction need explicit logic? |
| How do you dynamically register beans at runtime? | `BeanDefinitionRegistryPostProcessor` — rare, framework/plugin territory |
| Design patterns inside Spring itself? | Singleton, Factory, Proxy, Template Method, Front Controller — see table above |
| BeanFactory vs ApplicationContext, contexts, events? | `ApplicationContext` extends `BeanFactory` with eager singletons + events; events are sync unless `@Async` |
