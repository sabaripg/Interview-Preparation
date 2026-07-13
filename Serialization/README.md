# Serialization — Interview Preparation

> Serialization/deserialization fundamentals through the advanced trick questions senior interviewers actually reach for — cyclic references, Singleton-breaking, mixed Serializable/Externalizable hierarchies, and deep cloning. Every section has a worked Java example and an explicit pitfall.

## Contents

| # | Part | Covers |
|---|---|---|
| 1 | [Fundamentals](./Part1-Fundamentals.md) | `Serializable` marker interface, `serialVersionUID` (system vs manual), inheritance scenarios, blocking subclass serialization, `transient`/`static` field behavior, `Externalizable` |
| 2 | [Advanced](./Part2-Advanced.md) | Cyclic references & the object handle table, breaking Singleton + `readResolve()` fix, `readObject` vs `readResolve`, mixed Serializable/Externalizable hierarchy trick questions, deep cloning via serialization |

## How to use this

- Part 1 covers what most interviews actually ask; Part 2 is where senior-level follow-ups go.
- Cross-references: `core-java/Enums.md` (why enum singletons sidestep the `readResolve()` problem entirely), `core-java/Reflection.md` (why serialization's hook-method lookup mechanism costs what it costs).

---

*Last updated: July 2026*
