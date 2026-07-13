# Hibernate & JPA — Interview Preparation

> Spring Data JPA's repository abstraction through Hibernate's own core mechanics — locking, caching, the N+1 problem, entity lifecycle, and the ownership/cascade rules that cause silent persistence bugs. Every section has a worked Java example and an explicit pitfall.

## Contents

| # | Part | Covers |
|---|---|---|
| 1 | [JPA & Spring Data JPA](./Part1-JPA-Spring-Data.md) | CrudRepository vs JpaRepository, custom queries & repositories, findById vs deprecated getOne(), pagination, Query By Example, `@Embeddable`, optimistic vs pessimistic locking (with the precise "when is the lock actually held" answer), cascading OneToMany, query performance optimization |
| 2 | [Hibernate Core](./Part2-Hibernate-Core.md) | Spring Boot setup, get() vs load(), HQL, association default fetch types, the N+1 problem, Criteria API, entity states, Session vs SessionFactory, first/second-level cache, inheritance strategies, cascade vs inverse/mappedBy, hbm2ddl.auto, JDBC batching |

## How to use this

- Note that Spring Data JPA's `findById()`/`getReferenceById()` (Part 1) and Hibernate's own `get()`/`load()` (Part 2) are **the same underlying concept exposed through two different APIs** — interviewers sometimes ask both to see if you recognize that.
- Cross-references: `SpringAndSpringBoot/Part8-Spring-Data-JPA.md` for the Spring-layer configuration angle, `Multhithreading/Part4-Locks.md` for how pessimistic DB locking compares to in-JVM locking.

---

*Last updated: July 2026*
