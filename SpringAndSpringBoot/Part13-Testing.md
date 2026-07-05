# Part 13 — Testing

> The test pyramid for a Spring Boot app — unit, slice, and integration tests. Interview Q&A at the end.

## How to Test a Spring Boot Application

**Unit tests** — plain JUnit + Mockito, testing individual classes in isolation with mocked dependencies, no Spring context loaded at all. Fastest, and should be the majority of tests.

**Slice tests** — `@WebMvcTest` (loads only the web layer, mocks service dependencies), `@DataJpaTest` (loads only the JPA/repository layer against an in-memory or Testcontainers database), `@JsonTest`, etc. — load a minimal, focused subset of the application context, keeping tests fast while still testing framework integration for that layer.

**Integration tests** — `@SpringBootTest` loads the full application context (optionally with a real embedded/Testcontainers-backed database) for true end-to-end verification. Slower, used more sparingly for critical paths.

**`@MockBean`/`@SpyBean`** — replace specific beans with mocks within a loaded Spring context when needed.

**Testcontainers** — realistic integration tests against real database/message-broker instances in Docker, rather than relying on inconsistent in-memory substitutes (H2 behaving differently from real Postgres/MySQL is a common source of tests passing locally but failing in production-like conditions).

> ⚠️ **Pitfall — the test pyramid framing matters:** most tests should be fast unit tests with no Spring context; `@SpringBootTest` (full context) should be the exception, not the default, since it's slow and can turn a large test suite into a multi-minute CI bottleneck if overused. Mentioning Testcontainers specifically over H2-for-everything is a strong current-best-practice signal.

---

## Interview Q&A

**Q: How do you test a Spring Boot application?**
Covered above.
