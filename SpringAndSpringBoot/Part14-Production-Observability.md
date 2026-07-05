# Part 14 — Production & Observability

> Connection pooling, Actuator, custom health indicators, diagnosing latency regressions, and whether singleton beans are actually thread-safe. Interview Q&A at the end.

## Configuring Connection Pooling

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
```
**What it is:** Spring Boot uses **HikariCP** as the default connection pool since Boot 2.x (auto-configured if on the classpath, which it is by default via `spring-boot-starter-jdbc`/`data-jpa`).

**Key tuning parameters:** `maximum-pool-size` (should roughly match `(core_count * 2) + effective_spindle_count` per HikariCP's own sizing guidance, not an arbitrarily large number), `minimum-idle`, `connection-timeout`, `idle-timeout`, `max-lifetime`.

> ⚠️ **Pitfall:** "bigger pool = more throughput" is a common misconception — HikariCP's own documentation explicitly warns against oversized pools, since oversized pools can actually hurt throughput due to context-switching and DB-side contention. The right size is workload- and DB-capacity-dependent, not just "as large as possible."

## Spring Boot Actuator

**What it provides:** production-ready operational endpoints out of the box: `/actuator/health` (liveness/readiness, customizable with indicators), `/actuator/metrics` (JVM, HTTP, custom application metrics via Micrometer), `/actuator/info`, `/actuator/env`, `/actuator/loggers` (view/change log levels at runtime without redeploying).

**Integrating with external monitoring (Grafana):** Actuator exposes metrics via **Micrometer**, a vendor-neutral metrics facade ("the SLF4J of metrics") supporting multiple backends. Add `micrometer-registry-prometheus`, expose `/actuator/prometheus`, configure Prometheus to scrape it, then build Grafana dashboards against Prometheus as the data source. Enable relevant endpoints explicitly (`management.endpoints.web.exposure.include=health,metrics,prometheus`).

> ⚠️ **Pitfall:** always mention securing actuator endpoints in production — exposing `/actuator/env` or similar unsecured is a real, well-known security misconfiguration that can leak secrets/config values.

## Custom Health Indicators

```java
@Component
public class DownstreamServiceHealthIndicator implements HealthIndicator {
    public Health health() {
        boolean isUp = checkDownstreamConnectivity();
        return isUp ? Health.up().build()
                    : Health.down().withDetail("reason", "downstream unreachable").build();
    }
}
```
**What it does:** implement `HealthIndicator` (or extend `AbstractHealthIndicator`), register as a Spring bean — Actuator automatically picks it up and includes it in the aggregate `/actuator/health` response. Useful for surfacing dependency-specific health (database connectivity, downstream service reachability, disk space, queue depth) beyond Spring Boot's built-in indicators.

> ⚠️ **Pitfall:** if used for Kubernetes liveness/readiness probes, be careful that a custom indicator checking a *non-critical* downstream dependency doesn't cause the whole pod to be marked unhealthy/restarted for a transient issue in something the app could otherwise degrade gracefully around — differentiate liveness (is the app itself alive) from readiness (is it ready to serve traffic) health groups appropriately.

## Investigating a Sudden Latency Jump (<50ms → ~1s)

1. **Check recent deploys/config changes first** — correlate the timing of the regression with any recent release, config change, or infrastructure change. Often the fastest path to root cause.
2. **Take a thread dump** to check for lock contention, thread pool exhaustion, or threads blocked waiting on a slow downstream dependency.
3. **Check database query performance** — a missing index, table growth crossing a query-plan threshold, or a new N+1 pattern from a recent code change are extremely common causes.
4. **Check GC logs/metrics** for increased pause times (heap pressure from a leak or increased load).
5. **Check APM traces** (if available) to pinpoint which specific span/downstream call within the request is now slow.

> ⚠️ **Pitfall:** "check recent changes first" should be the very first instinct, not an afterthought — most production regressions correlate with *something* that changed. Jumping straight into deep profiling before checking "what changed recently" wastes time in the common case.

## Are Spring Singleton Beans Thread-Safe by Default?

**Answer: No.** Spring doesn't add any synchronization to a singleton bean — it's shared across every thread handling every request, so any **mutable instance state** on that bean is a genuine shared-state concurrency hazard, exactly like any other object shared across threads.

**Why this rarely bites in practice:** idiomatic Spring beans are **stateless** — a `@Service`/`@Repository` typically holds only injected, immutable dependencies (`final` fields set once via constructor injection) and derives all per-request data from method parameters, never from mutable instance fields.

**If a bean genuinely needs per-request/per-call mutable state:** the fix isn't manual synchronization on a shared singleton — it's `ThreadLocal` for thread-confined state, or `prototype` scope if a fresh instance per use is what's actually needed.

> ⚠️ **Pitfall:** the real senior answer isn't "beans aren't thread-safe, so synchronize them" — it's "design beans to be stateless in the first place, and this problem doesn't arise." Reaching for manual synchronization on a singleton service is usually a sign the bean shouldn't be holding that state at all.

---

## Interview Q&A

**Q: How would you configure connection pooling in a Spring Boot application?**
Covered above.

**Q: How can you integrate Spring Boot Actuator with external monitoring (e.g. Grafana)?**
Covered above.

**Q: Production server latency jumped from <50ms to ~1s — how would you investigate?**
Covered above.

**Q: Have you worked with Spring Boot Actuator?**
Covered above — have a specific example ready (e.g. "I built a custom health indicator that checked downstream service connectivity") rather than a purely textbook answer.

**Q: Can we create a custom health indicator in Spring Boot?**
Covered above.

**Q: Are Spring singleton beans thread-safe by default?**
Covered above.
