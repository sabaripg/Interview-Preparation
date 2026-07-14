# Java 8+ Functional Programming — Interview Preparation

> Stream API depth (creation, sorting, intermediate/terminal operations, a full `flatMap` deep-dive, primitive streams, `takeWhile`/`dropWhile`, collectors, parallel streams), lambdas and the built-in functional interfaces, method references, and a worked practice bank of real interview Stream questions against one running Employee dataset — the modern-Java fluency a 10-YOE interview expects beyond just "I use streams sometimes." Every section has a worked Java example and an explicit pitfall.

## Contents

| # | Part | Covers |
|---|---|---|
| 1 | [Streams API](./Part1-Streams-API.md) | Stream creation, sorting, intermediate vs terminal ops, laziness, a full `flatMap` deep-dive (why it exists + 4 concrete cases), primitive streams (`IntStream`/`DoubleStream`, `mapToInt`/`mapToDouble`, `sum`/`average`), `takeWhile`/`dropWhile`, Collectors (`groupingBy`/`partitioningBy`/`joining`/`mapping`), `Optional`, the `forEach`-side-effect anti-pattern, parallel streams and when not to use them |
| 2 | [Lambdas & Functional Interfaces](./Part2-Lambdas-Functional-Interfaces.md) | What a lambda actually compiles to (`invokedynamic`, not an anonymous class), `@FunctionalInterface`, the core `java.util.function` interfaces, effectively-final variable capture |
| 3 | [Method References](./Part3-Method-References.md) | The four kinds of method references, when they're clearer than a lambda and when they're not, ambiguous-overload gotchas |
| 4 | [Stream Interview Practice Bank](./Part4-Stream-Practice-Bank.md) | A full worked Employee-dataset bank: min/max lookups, department/company/location aggregations, `groupingBy`+`mapping`, `flatMap` over nested project lists, partitioning, and the N-th-highest-distinct-value pattern |

## How to use this

- Part 1 is the one most interviews actually probe — know it cold, especially the laziness model and the Collectors toolkit.
- Every part ends with an **Interview Q&A** section.
- Cross-references: `core-java/Generics.md` (functional interfaces are themselves generic types), `Multhithreading/` (parallel streams share a common ForkJoinPool with `parallelStream()` — see the Multhithreading notes on `ForkJoinPool.commonPool()` before reaching for parallel streams in production).

---

*Last updated: July 2026*
