# 📈 Part 14 — Production & Observability

> Neat, point-based format with callout boxes, tables, and icons. Interview Q&A at the end.

---

## 🏊 Configuring Connection Pooling

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
- Spring Boot uses **HikariCP** by default (auto-configured since Boot 2.x).

| Parameter | Purpose |
|---|---|
| `maximum-pool-size` | Upper bound — should roughly match `(core_count * 2) + effective_spindle_count`, per HikariCP's own guidance |
| `minimum-idle` | Minimum idle connections kept ready |
| `connection-timeout` | How long to wait for a connection before failing |
| `idle-timeout` / `max-lifetime` | Connection recycling |

> [!WARNING]
> "Bigger pool = more throughput" is a common misconception — HikariCP's own docs warn against oversized pools, since they can hurt throughput via context-switching and DB-side contention. Right-sizing depends on workload and DB capacity, not "as large as possible."

---

## 📡 Spring Boot Actuator

| Endpoint | Purpose |
|---|---|
| `/actuator/health` | Liveness/readiness, customizable with indicators |
| `/actuator/metrics` | JVM, HTTP, custom metrics via Micrometer |
| `/actuator/info` | App metadata |
| `/actuator/env` | Environment properties |
| `/actuator/loggers` | View/change log levels at runtime without redeploying |

**Integrating with Grafana:** Actuator exposes metrics via **Micrometer** (a vendor-neutral metrics facade). Add `micrometer-registry-prometheus`, expose `/actuator/prometheus`, scrape with Prometheus, visualize in Grafana.

> [!CAUTION]
> Always secure actuator endpoints in production — exposing `/actuator/env` unsecured is a real, well-known way to leak secrets/config values.

### Custom Health Indicators

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
- Implement `HealthIndicator`, register as a bean — Actuator picks it up automatically for `/actuator/health`.

> [!WARNING]
> For Kubernetes probes, don't let a custom indicator checking a *non-critical* downstream dependency mark the whole pod unhealthy for a transient issue — differentiate **liveness** (is the app alive) from **readiness** (is it ready for traffic).

---

## 🔍 Investigating a Sudden Latency Jump (<50ms → ~1s)

1. **Check recent deploys/config changes first** — the fastest path to root cause.
2. **Take a thread dump** — lock contention, thread pool exhaustion, or blocked-on-slow-downstream threads.
3. **Check database query performance** — missing index, table growth, or a new N+1 pattern.
4. **Check GC logs/metrics** — increased pause times from heap pressure.
5. **Check APM traces** — pinpoint the specific slow span.

> [!IMPORTANT]
> "Check recent changes first" should be the very first instinct, not an afterthought — most regressions correlate with *something* that changed.

---

## 🧵 Are Spring Singleton Beans Thread-Safe by Default?

**Answer: No.**
- Spring adds **no synchronization** to a singleton — it's shared across every thread handling every request.
- **Why this rarely bites in practice:** idiomatic beans are **stateless** — `final` injected dependencies, all per-request data from method parameters, never mutable instance fields.
- If genuine per-request mutable state is needed: `ThreadLocal` (thread-confined state) or `prototype` scope (fresh instance per use) — **not** manual synchronization on a shared singleton.

> [!IMPORTANT]
> The real senior answer isn't "synchronize the bean" — it's **"design beans to be stateless in the first place, and this problem doesn't arise."** Reaching for manual synchronization on a singleton is usually a sign the bean shouldn't hold that state at all.

---

## 📋 Interview Q&A

| Question | Short answer |
|---|---|
| Configuring connection pooling? | HikariCP defaults; don't oversize the pool |
| Integrating Actuator with Grafana? | Micrometer → Prometheus registry → `/actuator/prometheus` → Grafana |
| Latency jumped from <50ms to ~1s — how do you investigate? | Recent changes first, then thread dump, DB queries, GC, APM traces |
| Have you worked with Actuator? | Have a concrete example ready — e.g. a custom downstream health indicator |
| Custom health indicators? | Implement HealthIndicator; watch out for liveness vs readiness conflation |
| Are singleton beans thread-safe? | No — but stateless bean design avoids the problem entirely |
