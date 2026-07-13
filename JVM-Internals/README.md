# JVM Internals — Interview Preparation

> Architecture, class loading, JIT compilation, and garbage collection — with worked Java examples, pitfalls, and interview Q&A in every section.

## Contents

| # | Part | Covers |
|---|---|---|
| 1 | [Architecture & Class Loading](./Part1-Architecture-ClassLoading.md) | JDK/JRE/JVM, bytecode, the four architecture components, Heap vs Stack, class loading phases, classloader delegation, custom classloaders, JIT (C1/C2/AOT), tuning flags |
| 2 | [Garbage Collection](./Part2-Garbage-Collection.md) | Reachability & GC roots, minor vs major GC, the four core algorithms, why generational GC combines them, survivor space design, modern collector lineup (G1/ZGC/Shenandoah), Stop-The-World & safepoints, the four reference types |

## How to use this

- Every section follows the same pattern: what it is/does, a worked Java example with expected output, then an explicit `⚠️ Pitfall`.
- Each part ends with an **Interview Q&A** section for a fast pre-interview pass.
- Cross-references to `core-java/Exception-Handling.md` (OOM debugging, class-loading errors) and the `Multhithreading/` notes (lock guarantees, thread pools) are called out inline where the topics connect.

---

*Last updated: July 2026*
