# Part 8 — Spring Data & JPA

> EntityManager, persistence context, JdbcTemplate vs JPA, Lazy vs Eager fetching, composite keys. Interview Q&A at the end.

## The Purpose of EntityManager

**What it is:** the core JPA interface for interacting with the persistence context — managing the lifecycle of entities (persist, merge, remove, find), executing JPQL/native queries, and controlling transaction-scoped entity state (the first-level cache / persistence context that tracks managed entities and their changes for automatic dirty-checking at flush/commit time).

**Why you rarely touch it directly:** in Spring Data JPA, you rarely interact with `EntityManager` directly for basic CRUD — Spring Data repositories abstract it away — but it's the underlying mechanism repositories are built on, and you'd use it directly for complex custom queries or fine-grained persistence-context control.

> ⚠️ **Pitfall:** mention the persistence context / dirty-checking mechanism specifically — that's the conceptually important part, not just "it's how you talk to the database."

## How EntityManager Interacts with the Database

**What actually happens:** `EntityManager` doesn't talk to the database directly on every operation — it maintains an in-memory **persistence context** (first-level cache) of managed entities; changes are tracked but not immediately sent to the DB.

**When SQL actually gets sent:** at **flush time** — which happens automatically before a query execution that might be affected by pending changes, and always at transaction commit. This deferred-write behavior (via Hibernate as the typical JPA provider) batches/optimizes actual DB round trips.

> ⚠️ **Pitfall:** this deferred-flush behavior explains a lot of confusing JPA bugs ("why did my query not see this change I just made" — answer: it hadn't flushed yet, or flushed automatically because the query engine detected a potential dependency).

## JdbcTemplate vs Hibernate/JPA — When to Actually Choose JdbcTemplate

**What JdbcTemplate is:** a thin wrapper over raw JDBC that eliminates the usual connection/statement/resultset boilerplate but still has you write and reason about actual SQL directly — no persistence context, no automatic dirty-checking, no lazy loading, no entity lifecycle.

**What Hibernate/JPA gives you instead:** object-relational mapping, a persistence context with automatic change tracking, caching, and relationship management, at the cost of a learning curve around its behavior (N+1 problem, flush timing, `@Transactional` boundaries) and less direct control over the exact SQL executed.

**When to reach for JdbcTemplate:** the query is complex/reporting-style and you want full control over the exact SQL (window functions, complex joins ORMs handle awkwardly), you're doing bulk/batch operations where ORM overhead matters, or you specifically want to avoid persistence-context surprises for performance-critical read paths.

> ⚠️ **Pitfall:** "JDBCTemplate is faster than Hibernate" is an oversimplification often stated without justification — the real reason to reach for it is **control and predictability of the exact SQL**, not an inherent, universal performance win. A well-tuned JPA query with proper fetch strategies can match hand-written JDBC for many workloads.

## Lazy vs Eager Fetching for JPA Relationships

```java
@OneToMany(mappedBy = "user", fetch = FetchType.LAZY)  // default for collections
private List<Order> orders;
```

**Important distinction:** this is a **different concept** from Spring's bean lazy-initialization (`spring.main.lazy-initialization`, Part 3/4) — that controls *when Spring beans are constructed*; `FetchType` controls *when a JPA entity's related data is loaded from the database* relative to the owning entity.

**`FetchType.LAZY`** (default for `@OneToMany`/`@ManyToMany` collections) — related data is fetched only when the association is actually accessed, via a Hibernate-generated proxy — avoids loading data you don't need, but risks the classic **N+1 query problem** if you iterate and access the lazy association per-row without a fetch-join.

**`FetchType.EAGER`** (default for `@ManyToOne`/`@OneToOne`) — related data loaded immediately in the same query (or an immediate follow-up) — guarantees availability but risks over-fetching unused data.

> ⚠️ **Pitfall — best practice worth stating explicitly:** default to `LAZY` everywhere, and use an explicit `JOIN FETCH` JPQL query (or `@EntityGraph`) on the specific access paths that actually need the related data eagerly — rather than flipping the annotation to `EAGER`, which applies globally to every access and can't be tuned per query.

## Composite Keys in JPA

```java
@Embeddable
public class ProductRatingKey implements Serializable {
    private Long productId;
    private Long customerId;
}
@Entity
public class ProductRating {
    @EmbeddedId
    private ProductRatingKey id;
    @ManyToOne @MapsId("productId") @JoinColumn(name = "product_id")
    private Product product;
    @ManyToOne @MapsId("customerId") @JoinColumn(name = "customer_id")
    private Customer customer;
}
```
**What it does:** a composite (multi-column) primary key is modeled as a separate `@Embeddable` class implementing `Serializable`, then referenced via `@EmbeddedId` on the owning entity. `@MapsId` on each `@ManyToOne` ties that association's foreign key back to the corresponding field inside the composite key class — the FK columns double as both the relationship and part of the primary key, avoiding duplicate columns for the same value.

> ⚠️ **Pitfall:** forgetting `Serializable` on the `@Embeddable` key class is a common oversight — JPA providers rely on it for key-based operations (caching, certain provider internals) and some throw or warn without it.

---

## Interview Q&A

**Q: What is the purpose of EntityManager in JPA?**
Covered above.

**Q: How does EntityManager interact with the database in a Spring Boot application?**
Covered above.

**Q: JDBCTemplate vs Hibernate/JPA — when would you actually choose JDBCTemplate?**
Covered above.

**Q: Lazy vs Eager fetching for JPA relationships — not to be confused with Spring's bean lazy-initialization.**
Covered above.

**Q: What is a composite key in JPA, and how do you map it?**
Covered above.
