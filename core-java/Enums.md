# Enum

> What problem `enum` actually solves over constants, why it can't extend a class, constant-specific behavior, and the two enum-based design patterns (singleton, strategy) that come up most in senior interviews. Interview Q&A at the end.

## What Problem `enum` Solves

**Before `enum` (pre-Java 5):** developers modeled a fixed set of options with `int`/`String` constants.
```java
public static final int PENDING = 1;
public static final int APPROVED = 2;
public static final int REJECTED = 3;

void setStatus(int status) { ... }
setStatus(100); // compiles fine — nothing stops an invalid value
```
**What's actually wrong with this, precisely:**
1. **No type safety** — any `int` compiles, valid or not; the compiler can't help you.
2. **No namespace** — `PENDING` could collide with an unrelated constant named the same thing elsewhere.
3. **No behavior** — a constant is just a number; it can't carry logic.
4. **No compiler-enforced exhaustiveness** — nothing forces you to handle every valid value anywhere you switch on it.

```java
enum Status { PENDING, APPROVED, REJECTED }
void setStatus(Status status) { ... }
setStatus(100);  // compile-time error — not even a candidate
```
`enum` fixes all four: `Status` is its own type, `PENDING` lives in `Status`'s namespace (referenced as `Status.PENDING`), constants can have fields/methods, and modern `switch` expressions on enums can be checked for exhaustiveness by the compiler/IDE.

## Enums Are Real Classes — With Real Constraints

**What's actually happening under the hood:** each enum constant is a `public static final` instance of the enum type itself, and the compiler implicitly makes every enum `extend java.lang.Enum<E>` — which is why an enum can never `extend` anything else (Java has single inheritance, and the slot is already taken). An enum *can*, however, implement interfaces freely.

```java
interface Describable { String describe(); }

enum Status implements Describable {
    PENDING, APPROVED, REJECTED;
    public String describe() { return "Status: " + name(); }
}
```

> ⚠️ **Pitfall:** "enums can't extend a class" is the shallow answer — the precise reason (`enum` implicitly extends `java.lang.Enum<E>`, a `final` class, so the single-inheritance slot is used up) is what separates a memorized rule from real understanding, and it's exactly why enums *can* still implement any number of interfaces.

## Constant-Specific Behavior — the Genuinely Powerful Part

**What it does:** each enum constant can override an abstract method with its own implementation — effectively giving every value in the enum its own tiny anonymous subclass.

```java
enum PaymentStatus {
    SUCCESS { public boolean isFinal() { return true; } },
    FAILED  { public boolean isFinal() { return true; } },
    PENDING { public boolean isFinal() { return false; } };

    public abstract boolean isFinal();
}
```
Calling `PaymentStatus.PENDING.isFinal()` runs `PENDING`'s own override, not a shared `if/else` on `name()`. This is strictly better than a `switch` scattered across the codebase every time behavior needs to vary by constant — the behavior lives with the constant itself, and adding a new constant forces you to supply its behavior right there (the compiler won't let you forget the abstract method).

## Enum Singleton — the Recommended Way to Write a Singleton

```java
public enum ConfigManager {
    INSTANCE;
    private final Map<String, String> settings = new ConcurrentHashMap<>();
    public void set(String key, String value) { settings.put(key, value); }
    public String get(String key) { return settings.get(key); }
}

// usage
ConfigManager.INSTANCE.set("env", "prod");
System.out.println(ConfigManager.INSTANCE.get("env"));
```
**Output:**
```
prod
```
**Why this beats the classic `private static final INSTANCE` singleton pattern:** the JVM guarantees enum constant creation happens exactly once, and it's inherently thread-safe with zero synchronization code needed — no double-checked locking, no `volatile`, none of the ceremony Part 4 of the multithreading notes covers for the classic lazy singleton. It's also **serialization-safe by construction**: Java's serialization mechanism special-cases enums to serialize only the constant's `name()` and resolve it back to the same singleton instance on the other side — the classic singleton pattern needs an explicit `readResolve()` to get this same guarantee (see the Serialization notes), enum singletons get it for free.

> ⚠️ **Pitfall:** this is the *Effective Java* (Joshua Bloch) recommended way to implement a singleton in modern Java, specifically because it closes the two hardest-to-get-right classic singleton failure modes (broken by reflection calling a private constructor directly, and broken by deserialization creating a second instance) automatically, without the developer having to remember any defensive code.

## Enum Strategy Pattern

```java
public enum Operation {
    ADD { public int apply(int a, int b) { return a + b; } },
    SUBTRACT { public int apply(int a, int b) { return a - b; } },
    MULTIPLY { public int apply(int a, int b) { return a * b; } };

    public abstract int apply(int a, int b);
}

System.out.println(Operation.ADD.apply(3, 4));       // 7
System.out.println(Operation.MULTIPLY.apply(3, 4));  // 12
```
This is the same constant-specific-behavior mechanism as `PaymentStatus` above, applied deliberately as the Strategy design pattern — each constant *is* a strategy, selected by name instead of by constructing a separate strategy object per case.

## Iterating and the `Enum<E extends Enum<E>>` Declaration

```java
for (Status s : Status.values()) {
    System.out.println(s + " ordinal=" + s.ordinal());
}
```
`values()` is a compiler-synthesized static method (not inherited from `Enum`) returning a defensive copy of the constants array in declaration order — this is why `ordinal()` reflects *declaration order*, not any semantic ranking, and why reordering enum constants is a silent behavior change if code relies on ordinal comparisons.

**Why `Enum<E extends Enum<E>>` looks so strange:** it's a self-referential generic bound (`E` must be a subtype of `Enum<E>` itself) — this is what lets `Enum` provide type-safe methods like `compareTo(E)` that only accept the *same* enum type, rather than accepting any arbitrary `Enum`.

> ⚠️ **Pitfall — the ordinal trap:** never persist an enum's `ordinal()` to a database or serialize it as the source of truth for "which constant was this." Inserting a new constant in the middle of the declaration list shifts every ordinal after it, silently corrupting previously stored data. Persist `name()` (a stable string) instead, or an explicit custom code field.

---

## Interview Q&A

**Q: Why are enums better than `int`/`String` constants?**
Type safety (invalid values are compile errors, not runtime surprises), namespacing (`Status.APPROVED`, not a bare global constant), and the ability to carry fields/methods/behavior — none of which plain constants can do.

**Q: Can an enum extend another class? Why or why not?**
No — every enum implicitly extends `java.lang.Enum<E>`, a final class, and Java has single class inheritance. It can implement any number of interfaces, though, since that's unrestricted.

**Q: How would you implement a Singleton using enum, and why is this considered the best approach?**
`enum Singleton { INSTANCE; ... }` — the JVM guarantees single instantiation, thread-safety with no extra code, and automatic correctness under both reflection-based attacks and serialization/deserialization, which the classic singleton pattern needs explicit defensive code to achieve.

**Q: What's constant-specific method behavior, and when would you reach for it over a switch statement?**
Each enum constant can override an abstract method with its own body — behavior lives with the constant rather than being centralized in one big switch scattered (and duplicated) across the codebase. Reach for it when different constants need genuinely different logic, especially when you want the compiler to force every new constant to supply that logic.

**Q: Why shouldn't you persist `ordinal()` as a stored value?**
`ordinal()` reflects declaration order, which shifts if constants are reordered or a new one is inserted in the middle — silently corrupting any previously persisted ordinal-based data. Use `name()` or an explicit stable code field instead.
