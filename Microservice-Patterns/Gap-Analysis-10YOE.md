# Microservice Patterns — Gap Analysis for a 10-Year-Experience Interview

> Your source articles cover Service Discovery, Circuit Breaker, Load Balancing, and CQRS (with API Gateway filled in as a genuine gap in the first article, the database-sync mechanism filled in as a genuine gap in the CQRS article, and session affinity/sticky sessions added to Load Balancing after a real interview question). Saga — the pattern most candidates actually fail on — now has its own full write-up in **Part 6**, not just a summary here. That's a solid set, but a 10-YOE interview — especially given your Spring Boot / Kafka / AWS / Microservices background — expects you to reason about the *other* patterns that come up once a system has more than a couple of services talking to each other. Here's what's still not covered elsewhere in this folder, ranked by how likely it is to come up given your resume.

## 1. Saga Pattern — see Part 6 (high priority, directly relevant)

Moved to its own dedicated file: **`Part6-Saga-Pattern.md`** — covers local + compensating transactions, Choreography vs Orchestration, a full end-to-end Order/Inventory/Payment worked example with real compensation code, the isolation gap, stuck-saga/timeout handling, and a tricky-question bank. Given your Kafka background this is very likely to come up, and it's the single highest-value file in this folder to review closely.

## 2. Distributed Tracing (high priority, directly relevant)

**The problem:** a single user request can fan out across 5+ services. When it's slow or fails, which service in the chain is actually the problem? Logs scattered across 5 services with no shared identifier make this nearly impossible to answer quickly.

**The fix:** every request gets a **trace ID** at the entry point (often the API Gateway — see Part 4), propagated through every downstream call (via HTTP headers or Kafka message headers), so every log line and every span across every service can be correlated back to one request.

- **Legacy stack:** Spring Cloud Sleuth + Zipkin.
- **Current stack:** Sleuth is being superseded by **Micrometer Tracing** (Spring's own tracing facade) paired with **OpenTelemetry** as the instrumentation standard and Zipkin or Jaeger as the backend/visualization layer.

> ⚠️ Know the legacy-to-current naming shift here the same way you now know it for Hystrix→Resilience4j and Ribbon→Spring Cloud LoadBalancer — Sleuth is in the same "know it exists, but the ecosystem has moved on" category.

## 3. Config Server — externalized configuration (medium-high priority)

**The problem:** N services each with their own hardcoded `application.properties` means changing a shared value (a feature flag, a downstream URL, a rate limit) requires redeploying every service that uses it.

**The fix — Spring Cloud Config:** a dedicated Config Server serves configuration (often backed by a Git repo) to every service on startup, and can push refreshed config to running instances (via `/actuator/refresh` or a Spring Cloud Bus event) without a redeploy.

## 4. Service Mesh (medium priority, good to know exists)

**The problem:** circuit breaking, retries, mTLS, load balancing, and tracing can all be implemented as libraries inside each service (which is what Parts 1–4 of this folder describe) — but that means every service, in every language, has to correctly integrate and configure that library.

**The fix — a service mesh (Istio, Linkerd):** moves these cross-cutting concerns **out of application code** and into an infrastructure-level sidecar proxy (Envoy, typically) deployed alongside every service instance. The service code itself becomes simpler; the mesh handles retries/circuit-breaking/mTLS/observability transparently at the network layer.

> ⚠️ **Pitfall to mention if asked "when would you use a service mesh instead of Resilience4j":** a mesh is a bigger operational investment (you're now running and understanding Envoy/Istio) and makes sense once you have many services in multiple languages needing consistent resilience behavior — for a smaller, all-Java/Spring shop, in-process libraries like Resilience4j are often the more pragmatic choice. Don't present a mesh as a strict upgrade; it's a different tradeoff.

## 5. Idempotency in Distributed Systems (high priority, directly relevant given Kafka)

**The problem:** at-least-once delivery (the default in most messaging systems, including Kafka with typical consumer configurations) means a message can be processed **more than once** — a retried Kafka consumer, a retried HTTP call from a circuit breaker's retry policy (Part 2), a duplicate event from a Saga step.

**The fix:** design consumers/handlers to be **idempotent** — processing the same message twice produces the same end state as processing it once. Common techniques: a unique idempotency key per operation stored and checked before processing, upserts instead of inserts, or a "processed message IDs" dedup table with a unique constraint.

> Given your Kafka background this is a near-certain follow-up to any Saga or event-driven question — "what happens if that event gets delivered twice?"

## 6. Strangler Fig Pattern (lower priority, good context)

**The problem:** migrating a large monolith to microservices all at once is high-risk.

**The fix:** incrementally route specific pieces of functionality from the monolith to new microservices (often via the API Gateway acting as a router), shrinking the monolith piece by piece until it's fully "strangled" — the monolith and the new services coexist throughout the migration rather than a risky big-bang cutover.

## 7. Outbox Pattern (lower-medium priority, relevant if discussing Saga/Kafka/CQRS reliability)

**The problem:** a service that needs to both update its own database **and** publish an event about that update can't do both atomically by default — if it crashes between the DB commit and the Kafka publish, the event is lost, and other services never find out about a change that did happen.

**The fix:** write the event to an **outbox table** in the same local transaction as the actual business data change (so it's atomic by definition — same DB, same transaction), then a separate process (a polling publisher, or CDC via Debezium reading the DB's write-ahead log) reads the outbox table and reliably publishes to Kafka.

> This is the exact mechanism Part 5 (CQRS) points to for safely syncing a CQRS write model to its read model — one pattern, two use cases (Saga reliability and CQRS sync).

---

## Suggested priority if you're short on prep time

1. **Saga pattern + idempotency** (#1, #5) — highest odds of coming up given your Kafka/microservices background; these two are almost always asked together ("how do you handle a multi-step distributed operation, and what happens on duplicate delivery").
2. **Distributed tracing** (#2) — cheap to learn at a "know the current stack" level, commonly asked as a quick knowledge-currency check.
3. **Config Server** (#3) — straightforward, high probability of a quick factual question.
4. **Outbox pattern** (#7) — likely to surface as a natural follow-up if Saga/Kafka reliability or CQRS sync comes up (Part 5 already cross-references it).
5. **Service mesh + Strangler Fig** (#4, #6) — lower probability, but good to have a one-paragraph answer ready for "have you heard of X."
