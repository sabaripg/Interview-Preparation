# Part 2 — Hibernate Core

> Setup in Spring Boot, `get()` vs `load()`, HQL, the four association types and their default fetch behavior, the N+1 problem, Criteria API, entity states, Session vs SessionFactory, caching (first vs second level), inheritance strategies, cascade/inverse ownership, JSON infinite recursion on bidirectional relationships, and JDBC batching. Interview Q&A at the end.

## Hibernate Setup in Spring Boot

Mostly auto-configured — add `spring-boot-starter-data-jpa`, configure the datasource, and Spring Boot wires up `EntityManagerFactory`, Hibernate as the JPA provider, and transaction management with **no manual `SessionFactory` or XML** required.
```properties
spring.datasource.url=jdbc:mysql://localhost:3306/appdb
spring.datasource.username=root
spring.datasource.password=pass
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.database-platform=org.hibernate.dialect.MySQL8Dialect
```

## `session.get()` vs `session.load()`

`get()` **hits the database immediately** and returns `null` if the record doesn't exist — always a fully initialized object, appropriate when you're not sure the entity exists and need to touch its data right away. `load()` returns a **proxy** with no immediate query — Hibernate defers the actual `SELECT` until a property is first accessed, and if the record turns out not to exist, accessing the proxy throws `ObjectNotFoundException` rather than returning `null`. Use `load()` when you're confident the entity exists and want to benefit from lazy loading (e.g., you only need the ID to set up an association, and never actually touch any other field).

> ⚠️ **Pitfall:** this is the exact same underlying trade-off as `findById()` vs `getReferenceById()` in Spring Data JPA (Part 1) — Hibernate's `Session` API and Spring Data's repository API expose the same two behaviors under different names, and interviewers sometimes ask both to see if you recognize they're the same concept.

## HQL (Hibernate Query Language)

An object-oriented query language operating on **entity names and entity fields**, not table/column names — database-independent, and supports polymorphic queries (querying a superclass and getting results across all subclasses).
```java
from Employee e where e.department.name = :dept
```
**Case sensitivity, a genuinely easy trap:** keywords (`FROM`, `SELECT`, `WHERE`) are **not** case-sensitive, but entity and property names **are** case-sensitive (they must match your Java class/field names exactly, not your database column names).

**Commonly used `Query` methods:** `list()` returns all results; `executeUpdate()` runs an update/delete; `setFirstResult(n)`/`setMaxResults(n)` implement pagination manually; `setParameter(position/name, value)` binds parameters safely (see SQL injection section below).

## Associations and Their Default Fetch Types

```
| Association  | Default Fetch Type |
|---------------|---------------------|
| @ManyToOne    | EAGER               |
| @OneToMany    | LAZY                |
| @OneToOne     | EAGER               |
| @ManyToMany   | LAZY                |
```
> ⚠️ **Pitfall — this table is asked precisely because the defaults are counter-intuitive to some:** `@ManyToOne` and `@OneToOne` default to `EAGER`, while `@OneToMany`/`@ManyToMany` default to `LAZY` — the "to-one" side loads eagerly by default, the "to-many" side doesn't. Overriding `@ManyToOne` to `LAZY` explicitly is a common, deliberate performance optimization once a codebase matures, since the eager default can silently pull in more data than a given query path actually needs.

**Bidirectional relationships and JSON infinite recursion:** a bidirectional `@OneToOne`/`@OneToMany` means each side holds a reference back to the other — serializing either side to JSON naively recurses forever (`Player → PlayerProfile → Player → ...`). Fix with `@JsonManagedReference` on the owning side and `@JsonBackReference` on the inverse side — Jackson serializes only the managed side, breaking the cycle.
```java
class PlayerProfile {
    @OneToOne(mappedBy = "playerProfile")
    @JsonBackReference // this side is NOT serialized, breaking the cycle
    private Player player;
}
```

## The N+1 Problem

**What it is:** one query fetches N parent entities; then, because an association is lazily loaded and accessed per-parent in a loop, **N additional queries** fire — one per parent, to fetch each one's association separately. Example: 1 query for `users`, then N queries (one per user) for each user's `orders`.

**Fixes, in order of how often they're reached for:**
1. **`JOIN FETCH`** in JPQL — pulls the association into the same query.
2. **`@EntityGraph`** — declares a fetch plan for a specific query without changing the entity's global fetch type.
3. **`@BatchSize`** — batches the N follow-up queries into fewer, larger `IN (...)` queries instead of one-per-parent.

> ⚠️ **Pitfall:** N+1 is invisible in code review — the code looks correct, iterating a list and calling a getter. It only shows up as a real performance problem under load, in query logs or an APM tool, which is exactly why enabling `spring.jpa.show-sql=true` (or a proper SQL logging/monitoring setup) in staging/pre-prod is worth the noise — it's often the fastest way to actually *see* an N+1 pattern before a user reports slowness in production.

## Criteria API — Dynamic, Type-Safe Queries

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Employee> cq = cb.createQuery(Employee.class);
Root<Employee> emp = cq.from(Employee.class);
cq.where(cb.like(emp.get("fullName"), "%Hib%"));
List<Employee> results = em.createQuery(cq).getResultList();
```
Reach for Criteria specifically when the query's **shape itself** varies at runtime (an arbitrary combination of optional filters from a search form) — JPQL string-building for that scenario gets unwieldy and loses compile-time type checking fast.

## Entity States

**Transient** — a `new`-ed object with no session association and no database representation; if abandoned, Hibernate simply ignores it. **Persistent** — associated with an active session; Hibernate tracks changes and synchronizes them to the database on flush/commit; entered via `save()`/`persist()`, or by loading via `get()`/`load()`. **Detached** — was persistent, but the session closed (or the object was explicitly evicted); no longer tracked, so further changes aren't automatically saved. **Removed** — marked for deletion, pending flush.

**Reattaching a detached entity:**
```java
Entity managed = session.merge(detachedEntity); // returns a NEW managed instance -- the original stays detached
session.update(detachedEntity);                  // reattaches the SAME instance -- assumes it's not already persistent elsewhere
```
> ⚠️ **Pitfall:** `merge()` returns a *different* object than the one you passed in — continuing to use the original detached reference after calling `merge()` (instead of the returned managed instance) is a classic subtle bug, since further changes to the original object are silently untracked.

## `Session` vs `SessionFactory`

`SessionFactory` is a **thread-safe**, heavyweight object responsible for creating `Session`s — created **once per application** (or once per database, in a multi-datasource setup). `Session` is a **single-threaded**, lightweight object providing the actual CRUD operations — typically created **per request or per transaction**, opened, used, and closed within that single unit of work, never shared across threads.

## `hibernate.dialect` and `@GeneratedValue`

`hibernate.dialect` tells Hibernate which database-specific SQL variant to generate (`org.hibernate.dialect.PostgreSQLDialect`, `MySQL8Dialect`, etc.) — different databases have different SQL quirks Hibernate needs to account for.
```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```
- `AUTO` — Hibernate picks a strategy based on the dialect.
- `IDENTITY` — the database generates it (auto-increment column) — simple, but disables JDBC batch inserts for that entity (each insert needs its generated ID back before the next can proceed).
- `SEQUENCE` — uses a database sequence — Hibernate can pre-fetch sequence values, making it batch-insert-friendly, unlike `IDENTITY`.
- `TABLE` — a separate table simulates a sequence — portable across databases lacking real sequence support, but slower.

## First-Level Cache vs Second-Level Cache

**First-level** — scoped to a single `Session`, **enabled by default**, cannot be disabled. Loading/saving an entity within a session caches it; requesting the same entity again in that same session returns the cached instance without hitting the database. Cleared when the session closes.

**Second-level** — scoped to the `SessionFactory`, shareable **across multiple sessions**, **not enabled by default**, requires an explicit cache provider (Ehcache, Infinispan) and marking entities cacheable. Loading the same `Employee` in two entirely separate sessions can be served from the second-level cache instead of hitting the database twice, provided it's configured and the entity is cacheable.

## Inheritance Strategies

- **`SINGLE_TABLE`** — the whole hierarchy in one table, with a discriminator column. Fastest for queries (no joins), but sparse/nullable columns for subclass-specific fields across unrelated subclasses.
- **`JOINED`** — each class gets its own table, joined via foreign keys on query. Normalized, no wasted columns, but slower due to joins.
- **`TABLE_PER_CLASS`** — each **concrete** class gets its own complete table (no shared parent table). Avoids joins and nulls, but querying across the whole hierarchy needs `UNION`s, which is slow and awkward.

## Cascade and Inverse (`mappedBy`) — Two Genuinely Different Concepts

**Cascade** controls *which operations propagate* from parent to child (save/update/delete).
```java
@OneToMany(mappedBy = "department", cascade = CascadeType.ALL)
private List<Employee> employees;
```
`CascadeType.PERSIST`/`MERGE`/`REMOVE`/`REFRESH`/`ALL` — saving/deleting a `Department` correspondingly saves/deletes its `Employee`s.

**Inverse (`mappedBy`)** controls *which side owns the foreign key* — a completely orthogonal concern from cascading.
```java
@Entity class Employee {
    @ManyToOne @JoinColumn(name = "department_id") // OWNING side -- manages the actual FK column
    private Department department;
}
@Entity class Department {
    @OneToMany(mappedBy = "department") // INVERSE side -- a mirror, does NOT manage the FK
    private List<Employee> employees;
}
```
> ⚠️ **Pitfall — the mistake this rule exists to prevent:** if you set the association only on the inverse (`Department.employees`) side and never on the owning (`Employee.department`) side, **nothing gets persisted to the foreign key column at all** — Hibernate only writes the FK based on the owning side's state, regardless of what the inverse side's in-memory collection looks like. Always set the relationship from the owning side, or explicitly keep both sides in sync in application code.

## `hbm2ddl.auto` — What Each Value Actually Does

- `create` — drops and recreates every table on startup; **all existing data lost**.
- `create-drop` — same as `create`, plus drops tables again on shutdown; useful for tests only.
- `update` — updates the schema to match entity mappings, preserving existing data.
- `validate` — checks the schema matches the mappings, throws if not, **makes no changes**.
- `none` — Hibernate touches nothing.

> ⚠️ **Pitfall:** `create`/`update` in a real production environment is a well-known incident-waiting-to-happen — `create` destroys data outright, and even `update` can behave unexpectedly on some databases/versions for certain schema changes (column type narrowing, renames Hibernate can't safely infer). Production should run `validate` (or `none`), with schema changes managed through an explicit migration tool (Flyway/Liquibase) instead of Hibernate auto-DDL.

## SQL Injection, `final` Entities, and Named Queries

**SQL injection:** Hibernate's query APIs (HQL/JPQL, Criteria, JPA `Query`) use parameter binding under the hood, compiling down to JDBC `PreparedStatement` parameters — user input passed via `setParameter()` is always treated as *data*, never as SQL syntax. The risk reappears specifically when native queries are built via raw string concatenation instead of bound parameters.

**Why an entity class should never be `final`:** Hibernate implements lazy loading by generating a runtime **proxy subclass** of the entity. A `final` class can't be subclassed — lazy loading (and by extension, a large part of Hibernate's performance model) silently breaks or falls back to eager loading for that entity.

**Named queries** — precompiled and validated at `SessionFactory` startup (catching typos/errors immediately at boot rather than at first execution), and centralized/easier to maintain for complex queries. Trade-off: not customizable at runtime beyond supplied parameters (can't change sorting or structure dynamically), and changing one requires reloading the `SessionFactory` — can't be hot-swapped in a running application.

## Flush Time and JDBC Batching

**"Flushing"** means writing pending in-memory changes (INSERT/UPDATE/DELETE) to the actual database. Hibernate flushes: before a transaction commits (so the commit reflects the latest state), before executing a query that could be affected by pending changes (so the query sees consistent data), and whenever explicitly triggered via `session.flush()`.

**`hibernate.jdbc.batch_size`** controls how many statements of the same type get grouped into a single JDBC batch instead of executing individually:
1. Hibernate collects same-type SQL statements (e.g., all pending `Employee` inserts) in memory.
2. Adds them to an internal JDBC batch.
3. Once the configured batch size is reached, executes the whole batch in one round-trip.
4. A flush/commit before the batch fills executes whatever's accumulated immediately, even partially filled.

`batch_size = 0` (or unset) means no batching — every statement executes individually, increasing database round-trip overhead; acceptable for small operations or when debugging (individual statements are easier to see in logs), but a real throughput cost at scale. Note also that `GenerationType.IDENTITY` (above) effectively disables batching for inserts on that entity, regardless of the configured batch size, since each row's generated key must come back before the next insert can be batched.

---

## Interview Q&A

**Q: `session.get()` vs `session.load()` — what's the actual difference, and when would you pick each?**
`get()` hits the database immediately and returns `null` if missing; `load()` returns a lazy proxy that only queries on first property access and throws `ObjectNotFoundException` if the row turns out not to exist. Use `get()` when existence is uncertain and you need the data now; `load()` when you're confident it exists and want to defer the actual query.

**Q: What's the N+1 problem, and what are the three standard fixes?**
One query for parent entities, then one additional query per parent for a lazily-loaded association accessed in a loop. Fix with `JOIN FETCH` (pull the association into the same query), `@EntityGraph` (declare a per-query fetch plan), or `@BatchSize` (batch the follow-up queries into fewer, larger ones).

**Q: Cascade vs Inverse/`mappedBy` — are these the same concept?**
No — cascade controls which *operations* (save/delete/etc.) propagate from parent to child; `mappedBy`/inverse controls which *side owns the foreign key* in a bidirectional relationship. Setting a bidirectional association only from the inverse side, with the owning side never updated, persists nothing to the actual FK column.

**Q: Why can't an entity class be `final` in Hibernate?**
Hibernate implements lazy loading via a runtime-generated proxy subclass of the entity — a `final` class can't be subclassed, so lazy loading breaks (or silently falls back to eager) for that entity.

**Q: Why is `hbm2ddl.auto=update` risky in production, even though it doesn't drop tables like `create` does?**
It can behave unpredictably for schema changes Hibernate can't safely infer from entity mappings (column narrowing, renames), and it bypasses any real migration history/rollback story. Production should use `validate`/`none` with schema changes managed by an explicit migration tool like Flyway or Liquibase.
