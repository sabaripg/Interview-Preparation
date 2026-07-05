# 🗄️ Part 8 — Spring Data & JPA

> Neat, point-based format with callout boxes, tables, and icons. Interview Q&A at the end.

---

## 🧠 The Purpose of EntityManager

- The core JPA interface for managing entity lifecycle (persist, merge, remove, find), executing JPQL/native queries, and controlling the **persistence context** (first-level cache tracking managed entities for automatic dirty-checking).
- In Spring Data JPA, you rarely touch it directly for basic CRUD — repositories abstract it away — but it's the underlying mechanism they're built on.

> [!IMPORTANT]
> Mention the persistence context / dirty-checking mechanism specifically — that's the conceptually important part, not just "it's how you talk to the database."

---

## 🔄 How EntityManager Interacts with the Database

- Doesn't hit the DB on every operation — maintains an in-memory **persistence context**; changes are tracked but not immediately sent.
- Actual SQL is sent at **flush time** — automatically before a query that might be affected by pending changes, and always at transaction commit.

> [!TIP]
> This deferred-flush behavior explains a lot of confusing JPA bugs — "why didn't my query see this change I just made" — answer: it hadn't flushed yet, or flushed automatically because the engine detected a dependency.

---

## ⚖️ JdbcTemplate vs Hibernate/JPA

| | `JdbcTemplate` | Hibernate/JPA |
|---|---|---|
| What you get | Thin wrapper over raw JDBC — no persistence context, no dirty-checking, no lazy loading | ORM, persistence context, automatic change tracking, caching, relationship management |
| Control over SQL | Full, direct control | Less direct — behavior depends on fetch strategies |
| Learning curve | Low | Real — N+1 problem, flush timing, `@Transactional` boundaries |

**When to reach for JdbcTemplate:** complex/reporting-style queries needing full SQL control, bulk/batch operations where ORM overhead matters, or avoiding persistence-context surprises on performance-critical read paths.

> [!WARNING]
> "JDBCTemplate is faster than Hibernate" is an oversimplification — the real reason to reach for it is **control and predictability of the exact SQL**, not a universal performance win. A well-tuned JPA query with proper fetch strategies can match hand-written JDBC.

---

## ⚡ Lazy vs Eager Fetching for JPA Relationships

```java
@OneToMany(mappedBy = "user", fetch = FetchType.LAZY)  // default for collections
private List<Order> orders;
```

> [!NOTE]
> This is a **different concept** from Spring's bean `spring.main.lazy-initialization` — that controls when Spring beans are constructed; `FetchType` controls when a JPA entity's related data loads from the database.

| FetchType | Default for | Behavior | Risk |
|---|---|---|---|
| `LAZY` | `@OneToMany`/`@ManyToMany` | Loaded only when accessed, via a Hibernate proxy | **N+1 query problem** if accessed per-row in a loop |
| `EAGER` | `@ManyToOne`/`@OneToOne` | Loaded immediately with the owning entity | Over-fetching unused data |

> [!TIP]
> **Best practice:** default to `LAZY` everywhere, and use an explicit `JOIN FETCH` JPQL query (or `@EntityGraph`) on the specific access paths that need eager loading — rather than flipping to `EAGER` globally, which can't be tuned per query.

---

## 🔑 Composite Keys in JPA

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
- A composite (multi-column) key is modeled as an `@Embeddable` class implementing `Serializable`, referenced via `@EmbeddedId`.
- `@MapsId` ties each `@ManyToOne`'s foreign key back to the corresponding composite-key field, avoiding duplicate columns.

> [!WARNING]
> Forgetting `Serializable` on the `@Embeddable` key class is a common oversight — JPA providers rely on it for key-based operations, and some throw/warn without it.

---

## 📋 Interview Q&A

| Question | Short answer |
|---|---|
| Purpose of EntityManager? | Manages entity lifecycle + persistence context for dirty-checking |
| How EntityManager interacts with the DB? | Deferred writes — actual SQL sent at flush time, not immediately |
| JdbcTemplate vs Hibernate/JPA — when to choose JdbcTemplate? | Complex SQL control, bulk ops, avoiding persistence-context surprises |
| Lazy vs Eager fetching? | LAZY default for collections (risk: N+1); EAGER default for single associations (risk: over-fetch) |
| What is a composite key in JPA? | `@Embeddable` + `@EmbeddedId` + `@MapsId` on each FK relationship |
