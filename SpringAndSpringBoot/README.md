# Spring / Spring Boot — Interview Preparation

A complete, corrected, and expanded Spring/Spring Boot interview guide — basic fundamentals through advanced topics, each part with What/Why/When explanations, worked code, pitfalls, and interview Q&A. Sourced and cross-checked against your Notion Chapter 14 (103 questions).

## 📚 Contents

| # | Part | Covers |
|---|---|---|
| 1 | [Core Container Fundamentals](./Part1-Core-Container-Fundamentals.md) | Why Spring, what a Bean is, ApplicationContext, stereotype annotations, @Configuration vs @Bean, design patterns in Spring |
| 2 | [Dependency Injection](./Part2-Dependency-Injection.md) | @Autowired vs @Inject, constructor/setter/field injection, @Primary vs @Qualifier, circular dependencies |
| 3 | [Bean Scope & Lifecycle](./Part3-Bean-Scope-Lifecycle.md) | Singleton vs prototype, all scopes, full lifecycle, eager vs lazy, the @PostConstruct + @Transactional gotcha |
| 4 | [Auto-Configuration & Starters](./Part4-AutoConfiguration-Starters.md) | Bootstrapping, @Profile vs @Conditional, how auto-config works, custom starters, startup internals, reducing startup time |
| 5 | [AOP, Proxies & Transactions](./Part5-AOP-Proxies-Transactions.md) | AOP fundamentals, custom annotations, @Transactional internals and every silent-failure gotcha, propagation types |
| 6 | [Web Layer: MVC, CORS](./Part6-Web-Layer-MVC-CORS.md) | DispatcherServlet, @RestController vs @Controller, Filters vs Interceptors, CORS end-to-end, WebFlux |
| 7 | [REST API Design](./Part7-REST-API-Design.md) | Large datasets, idempotency, HTTP status codes, PUT vs PATCH, ModelMapper, global exception handling, API docs |
| 8 | [Spring Data & JPA](./Part8-Spring-Data-JPA.md) | EntityManager, persistence context, JdbcTemplate vs JPA, Lazy vs Eager fetching, composite keys |
| 9 | [Configuration & Properties](./Part9-Configuration-Properties.md) | @Value, @ConfigurationProperties, YAML vs properties |
| 10 | [Async & Scheduling](./Part10-Async-Scheduling.md) | @Async gotchas, @Scheduled types, thread pool configuration |
| 11 | [Messaging: RabbitMQ](./Part11-Messaging-RabbitMQ.md) | Producer-consumer, acknowledgment, DLQs, ordering, scaling, delay, exactly-once, RPC, security, monitoring |
| 12 | [Messaging: Kafka](./Part12-Messaging-Kafka.md) | Event-driven communication, the same pattern applied with Kafka |
| 13 | [Testing](./Part13-Testing.md) | Unit vs slice vs integration tests, Testcontainers, the test pyramid |
| 14 | [Production & Observability](./Part14-Production-Observability.md) | Connection pooling, Actuator, custom health indicators, diagnosing latency, singleton thread-safety |

## How to use this

- Start at Part 1 and work forward if you're building fundamentals up.
- Jump straight to a part if you're revising a specific topic before an interview.
- Every part ends with an **Interview Q&A** section — good for a fast pre-interview pass.
- Every concept leads with **what it does**, **why it matters**, and **when to use it** before any code.

---

*Last updated: July 2026*
