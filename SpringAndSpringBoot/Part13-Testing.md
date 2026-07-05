# 🧪 Part 13 — Testing

> Neat, point-based format with callout boxes, tables, and icons. Interview Q&A at the end.

---

## 🏗️ How to Test a Spring Boot Application

| Test type | What it loads | Speed | Use for |
|---|---|---|---|
| **Unit tests** | No Spring context — plain JUnit + Mockito | Fastest | The majority of tests — isolated class logic |
| **Slice tests** | A minimal, focused subset (`@WebMvcTest`, `@DataJpaTest`, `@JsonTest`) | Fast | Testing one layer's framework integration |
| **Integration tests** | Full context (`@SpringBootTest`) | Slowest | True end-to-end verification of critical paths |

- `@MockBean`/`@SpyBean` — replace specific beans with mocks within a loaded Spring context.
- **Testcontainers** — realistic integration tests against real DB/broker instances in Docker, instead of inconsistent in-memory substitutes (H2 behaving differently from real Postgres/MySQL is a common cause of "works locally, fails in prod-like conditions").

> [!IMPORTANT]
> **The test pyramid framing matters:** most tests should be fast unit tests with no Spring context. `@SpringBootTest` should be the exception, not the default — it's slow and can turn a large suite into a multi-minute CI bottleneck if overused. Mentioning Testcontainers over H2-for-everything is a strong current-best-practice signal.

---

## 📋 Interview Q&A

| Question | Short answer |
|---|---|
| How do you test a Spring Boot application? | Unit tests (majority) → slice tests → integration tests (@SpringBootTest, sparingly); Testcontainers over H2 |
