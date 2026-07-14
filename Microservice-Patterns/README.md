# Microservice Patterns — Interview Preparation

Deepened from your Medium article on Microservice Patterns — Service Discovery, Circuit Breaker, and Load Balancing — plus **API Gateway**, which your article's own table of contents promised but never actually covered (a genuine gap, filled in here). Every part has a runnable Spring Boot example, an explicit pitfall, and a diagram where the concept is genuinely visual. Legacy-vs-modern library status (Hystrix, Ribbon, Zuul are all in maintenance mode) is called out everywhere it matters — a 10-YOE interviewer will notice if you don't know this.

## 📚 Contents

| # | Part | Covers |
|---|---|---|
| 1 | [Service Discovery](./Part1-Service-Discovery.md) | Client-side vs server-side discovery, Eureka server/client setup, self-preservation mode, AP vs CP tradeoff |
| 2 | [Circuit Breaker](./Part2-Circuit-Breaker.md) | Closed/Open/Half-Open states, Hystrix (legacy) vs Resilience4j (modern), Bulkhead isolation |
| 3 | [Load Balancing](./Part3-Load-Balancing.md) | Client-side vs server-side, algorithms, Ribbon (legacy) vs Spring Cloud LoadBalancer (modern), **session affinity / sticky sessions** |
| 4 | [API Gateway](./Part4-API-Gateway.md) | Spring Cloud Gateway routing/predicates/filters, Zuul (legacy), cross-cutting concerns |
| 5 | [CQRS](./Part5-CQRS.md) | Commands vs Queries, materialized views, the real database-sync mechanisms (event sourcing, CDC, outbox), the dual-write trap, eventual consistency, when it's overkill |
| 6 | [Saga Pattern](./Part6-Saga-Pattern.md) | Local + compensating transactions, Choreography vs Orchestration, an end-to-end Order/Inventory/Payment worked example, the isolation gap, and why most candidates fail this question |
| — | [Gap Analysis — 10 YOE](./Gap-Analysis-10YOE.md) | Patterns a senior interview expects that the source articles don't cover: Config Server, distributed tracing, service mesh, and more |

## How to use this

- Start at Part 1 if building this topic up from scratch; jump to any part to revise before an interview.
- Every part ends with an **Interview Q&A** section.
- Every legacy library mentioned (Hystrix, Ribbon, Zuul) is paired with its modern replacement — know both, since interviewers may ask about either depending on what a legacy codebase you inherited was using.
- Diagrams referenced inline (`images/*.svg`) sit one folder up from these files.

---

*Last updated: July 2026*
