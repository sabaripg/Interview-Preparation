# Part 1 — Core Container Fundamentals

> What Spring actually gives you, what a Bean is, where beans live, and the stereotype annotations. Interview Q&A at the end.

## Why Do We Need Spring?

**What it does:** a Dependency Injection / IoC container that manages object creation and wiring, decoupling classes from directly instantiating their collaborators.

**Why it matters:** decoupled construction makes code more testable (easy to swap in mocks) and maintainable. Beyond DI itself, Spring reduces boilerplate for cross-cutting concerns (transactions, security, AOP-based logging) via declarative annotations instead of hand-written plumbing, and provides consistent, well-tested abstractions over lower-level APIs (`JdbcTemplate` over raw JDBC, JMS, REST clients). Spring Boot specifically adds auto-configuration and opinionated defaults, cutting setup time from days to minutes for common application types.

**When you'd explain it this way:** whenever asked "why Spring" or "why not just write plain Java" — lead with **why DI/IoC matters** (testability, decoupling), not "it's a framework for building Java apps." That's the foundational value proposition everything else builds on.

## What Is a Spring Bean?

**What it is:** any object whose lifecycle (creation, configuration, wiring, destruction) is managed by the Spring IoC container, rather than being manually instantiated with `new` by application code.

**Why the container matters here:** "managed by the container" is the defining characteristic — not the object itself. A plain Java object becomes a "bean" specifically because Spring owns its lifecycle.

**When beans get registered:** via `@Component`/`@Service`/`@Repository`/`@Controller` (component scanning) or explicitly via `@Bean` methods in a `@Configuration` class. Beans are typically singletons by default (one instance per container — see Part 3), retrieved from the `ApplicationContext` and injected wherever needed via `@Autowired`/constructor injection.

## Where Are All Beans Stored? (ApplicationContext)

**What it is:** the `ApplicationContext` is Spring's IoC container — the "bean warehouse" — holding bean definitions and, for singleton-scoped beans, the actual instantiated instances, in an internal `BeanFactory` (the underlying registry `ApplicationContext` builds on and extends with enterprise features: event publishing, internationalization, environment abstraction).

**Why it works this way:** beans are looked up by name/type when needed for injection; the container manages the full lifecycle — instantiation, dependency injection, initialization callbacks (`@PostConstruct`), and eventual destruction (`@PreDestroy`).

> ⚠️ **Pitfall:** distinguish `BeanFactory` (basic container, lazy-loads beans) from `ApplicationContext` (superset, eagerly initializes singleton beans by default, adds enterprise features) if asked to go deeper — `ApplicationContext` is what virtually all real applications use.

## What Is a Spring Container?

**What it is:** the runtime environment that creates, configures, wires, and manages the complete lifecycle of beans — implemented concretely as `BeanFactory` (base) and `ApplicationContext` (the practical, feature-rich interface almost always used).

**When to use which term:** "container" and "ApplicationContext" are effectively synonyms in casual usage — a precise answer distinguishes the abstract concept (container/IoC) from the concrete interface (`ApplicationContext`).

## The Stereotype Annotations — @Component, @Repository, @Service, @Controller

**What they are:** all four mark a class as a Spring-managed bean, discoverable via component scanning. `@Repository`, `@Service`, and `@Controller` are all meta-annotated with `@Component`, so they're all "components" at the core mechanism level.

| Annotation | What it's for | Extra behavior beyond @Component |
|---|---|---|
| `@Component` | generic stereotype for any Spring-managed bean | none |
| `@Repository` | data-access layer | automatic **exception translation** — JDBC/JPA-specific exceptions get wrapped into Spring's unified `DataAccessException` hierarchy |
| `@Service` | business/service layer | purely semantic — signals intent, no extra framework behavior |
| `@Controller` | Spring MVC web controller | methods return view names by default (or JSON with `@ResponseBody`/`@RestController`) |

> ⚠️ **Pitfall:** `@Repository`'s exception translation is the one concrete *functional* difference among these annotations — the rest are semantic/organizational. Know this distinction, since it's the actual technical answer interviewers are checking for.

## @Configuration vs @SpringBootConfiguration

**What they are:** `@Configuration` is the core Spring annotation marking a class as a source of `@Bean` definitions (Java-based configuration, replacing XML config). `@SpringBootConfiguration` is a Spring Boot-specific annotation, itself meta-annotated with `@Configuration`, used internally to mark the main application class (part of what `@SpringBootApplication` bundles, alongside `@EnableAutoConfiguration` and `@ComponentScan`).

**Why it exists separately:** mainly so testing frameworks can reliably find "the one" primary configuration class in an application via a dedicated marker, distinct from any of the potentially many other `@Configuration` classes in the app.

> ⚠️ **Pitfall:** most engineers never use `@SpringBootConfiguration` directly (it's applied automatically via `@SpringBootApplication`) — its main practical purpose is enabling test-context discovery, not something you'd typically add manually.

## @Component vs @Bean

**What they're for:** `@Component` is a class-level annotation applied to a class you own the source of; discovered via component scanning, the container instantiates it directly. `@Bean` is a method-level annotation inside a `@Configuration` class, used when you need to register a bean you *don't* own the source for (third-party library classes), need conditional/programmatic construction logic, or need multiple differently-configured instances of the same class as separate beans.

**When to use which:** `@Component` for your own classes with a simple, no-args-or-injectable constructor. `@Bean` when you need explicit construction logic or don't control the class's source.

## Dynamic Bean Registration at Runtime (Advanced)

**What it does:** lets you register bean definitions programmatically instead of via annotations, before the container fully initializes.

```java
@Component
public class DynamicBeanRegistrar implements BeanDefinitionRegistryPostProcessor {
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        registry.registerBeanDefinition("dynamicBean",
            BeanDefinitionBuilder.genericBeanDefinition(MyBean.class).getBeanDefinition());
    }
}
```
Implement `BeanDefinitionRegistryPostProcessor` (or the simpler `BeanFactoryPostProcessor`). Alternative: `GenericApplicationContext.registerBean()` for a manually-bootstrapped context, or `ApplicationContext.getAutowireCapableBeanFactory()` for imperative registration in more dynamic/plugin-style scenarios.

**When you'd actually use this:** multi-tenant applications registering tenant-specific beans, or plugin systems loading beans based on discovered modules at startup.

> ⚠️ **Pitfall:** this is a genuinely advanced/rare technique — be honest it's uncommon in typical application code and reserved for framework-like or plugin-architecture scenarios, not everyday practice.

## Design Patterns Inside Spring Itself

**Singleton** — singleton-scoped beans. **Factory** — `BeanFactory` and `@Bean` factory methods. **Prototype** — prototype-scoped beans. **Proxy** — the AOP mechanism behind `@Transactional`/`@Async`/`@Cacheable` (JDK dynamic proxies or CGLIB). **Template Method** — `JdbcTemplate`, `RestTemplate`, `RabbitTemplate` (fixed algorithm skeleton, customizable steps). **Front Controller** — `DispatcherServlet`. **MVC** — Spring MVC itself. **DAO** — Spring's data-access abstraction layer.

> ⚠️ **Pitfall:** naming `DispatcherServlet` as Front Controller and the `@Transactional`/`@Async` machinery as Proxy pattern shows you connect Spring's internals to general design-pattern vocabulary, not just reciting Spring-specific terms in isolation.

## BeanFactory vs ApplicationContext, Parent-Child Contexts, and Events — How They Tie Together

`BeanFactory` is the root container interface — lazy by default, minimal features, rarely instantiated directly in modern code. `ApplicationContext` extends it and adds eager singleton instantiation (fail-fast at startup), `MessageSource` (i18n), `Environment`/profile access, and `ApplicationEventPublisher` — this is what Spring Boot always uses under the hood.

**Parent-child contexts:** a parent context's beans are visible to a child context, but never the reverse — the classic pre-Boot Spring MVC pattern split a root `ApplicationContext` (services/repositories) from a child `WebApplicationContext` (controllers). Spring Boot usually collapses this into one context now, but it explains historical `@ComponentScan` placement issues.

**Spring's event model** — `ApplicationEventPublisher.publishEvent()` + `@EventListener` — lets beans communicate without direct coupling:
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
> ⚠️ **Pitfall:** events are **synchronous by default** — `publishEvent()` blocks until every listener finishes, and an exception in one listener propagates back to the publisher. Assuming application events are inherently async is a common mistake — add `@Async` (with `@EnableAsync`) on listeners that shouldn't block or fail the original operation.

---

## Interview Q&A

**Q: Why do we need Spring?**
Covered above.

**Q: What exactly is a Spring Bean?**
Covered above.

**Q: Where are all beans stored?**
Covered above under ApplicationContext.

**Q: What exactly is a Spring Container?**
Covered above.

**Q: Difference between @Component, @Repository, @Service, and @Controller?**
Covered above.

**Q: Difference between @Configuration and @SpringBootConfiguration?**
Covered above.

**Q: Difference between @Component and @Bean?**
Covered above.

**Q: How do you dynamically register beans at runtime in Spring Boot?**
Covered above.

**Q: Name some design patterns used in the Spring Framework itself.**
Covered above.

**Q: BeanFactory vs ApplicationContext, parent-child contexts, and Spring's event system — what ties them together?**
Covered above.
