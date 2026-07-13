# ☕ Chapter 2 — Core Java (Topic-wise)

> Segregated from the original 59-question flat list into thematic sections for easier revision.
> **Bold** = key answer point. `⚠️ Pitfall` = interview trap / common mistake.

**Deep-dive companion files in this folder** (same style — explanation, worked Java code, pitfalls):
[Reflection](./Reflection.md) · [Enum](./Enums.md) · [Generics](./Generics.md) · [Exception Handling](./Exception-Handling.md)

---

## 1. JVM Startup, Main Method & Program Execution

### Q1. Explain `System.out.println`
- `System.out` is a **`static final PrintStream`** field on `java.lang.System`, initialized by the JVM at startup wrapping standard output.
- `println()` writes the string representation + platform line separator, and **auto-flushes on newline**.
- Ultimately routes through `FileOutputStream` → file descriptor 1 (stdout).

> ⚠️ **Pitfall:** `System.out`/`err`/`in` **can be reassigned at runtime** via `System.setOut()` — relevant for redirecting output in tests.

### Q2. Why is `System.out` considered thread-safe?
- `PrintStream` methods are **`synchronized` on the stream instance** — a single `println()` call won't have its output garbled/interleaved.
- Does **not** guarantee ordering *between* separate calls from different threads.

> ⚠️ **Pitfall:** "Thread-safe" here = won't corrupt a single write, **not** deterministic multi-line ordering across threads.

### Q3. Can we execute a program without a `main()` method?
- Pre-Java 7: a static initializer block + `System.exit()` could run before the "no main" error — **closed in Java 7+**.
- Modern Java: `public static void main(String[] args)` is mandatory.
- Java 21+ preview "implicit classes/instance main methods" reduces boilerplate only, not the requirement itself.

> ⚠️ **Pitfall:** Don't state the static-initializer trick as still working today.

### Q29. What happens when `main()` is run?
- JVM starts → loads main class (bootstrap → platform → application loader) → runs **static initializers in declaration order** → invokes `main(String[])` on the main thread.
- JVM stays alive as long as any **non-daemon thread** runs; shutdown hooks run before exit.

> ⚠️ **Pitfall:** `main` returning ≠ JVM exiting immediately if other non-daemon threads are alive.

### Q30. What happens under the hood after `System.out.println("Hello")`?
- **Compile** (`javac` → bytecode) → **Class Load** (classloader + verify + prepare/init) → **Execute** (interpret, then JIT hot paths to native code) → `println` writes via `PrintStream` → OS stdout fd.

> ⚠️ **Pitfall:** Structure the answer in clear stages — interviewers check mental-model organization, not just facts.

### Q6. Can `throws` be used with `run()` or `main()`?
- `main`: **yes** — `throws Exception` legal; uncaught exception → JVM prints stack trace, exits non-zero.
- `Runnable.run()`: **no** — interface declares no `throws`; overriding method can't widen checked exceptions.

> ⚠️ **Pitfall:** Ties directly to the "can't widen checked exceptions when overriding" OOPS rule.

### Q7. What happens if you call `Thread.wait()` in main?
- `wait()` is on `Object`, requires holding the monitor lock.
- No lock held → `IllegalMonitorStateException`.
- Held correctly → **the calling thread** (main, here) blocks until notified/interrupted.

> ⚠️ **Pitfall:** `someThread.wait()` doesn't make "the thread object" wait specially — it makes **whichever thread calls it** wait on that object's monitor.

### Q4. JDK 14-compiled program run on JDK 8 runtime?
- **`UnsupportedClassVersionError`** at class-loading — bytecode version embedded in `.class` must be ≤ JVM's supported version.

> ⚠️ **Pitfall:** Mention `--release`/`-source`/`-target` flags as the practical fix.

### Q5. Java 8-compiled app on JDK 11 runtime?
- Generally **yes**, backward compatible.
- Caveats: removed APIs (JAXB/Java EE modules removed from JDK 11+), internal API (`sun.*`) restrictions.

> ⚠️ **Pitfall:** Don't say "always works unconditionally" — name the JAXB/Java EE removal explicitly.

---

## 2. Object Class, Equality & Cloning

### Q9. What is the `Object` class?
- **Root of the class hierarchy** — every class implicitly extends it.
- Provides `equals`, `hashCode`, `toString`, `getClass`, `wait`/`notify`/`notifyAll`.

> ⚠️ **Pitfall:** None of `Object`'s methods are abstract — default working implementations exist for every class.

### Q10. Important methods in `Object`
- `equals(Object)`, `hashCode()` — must be overridden **together, consistently**.
- `toString()`, `getClass()`, `clone()` (protected, needs `Cloneable`), `wait/notify/notifyAll`, `finalize()` (**deprecated since Java 9**).

> ⚠️ **Pitfall:** Listing `finalize()` without the deprecation caveat signals outdated knowledge.

### Q11. Default `hashCode()` of an object
- Implementation-specific, often identity-related — **not guaranteed to be the literal memory address**.
- Consistent for the same object across its lifetime.

> ⚠️ **Pitfall:** Don't state "it's literally the memory address" as absolute fact.

### Q42. `==` vs `.equals()` — the real difference
- `==` on primitives compares value; on references compares **identity**.
- `.equals()` defines **logical equality** (defaults to identity unless overridden).
- Without override, `.equals()` behaves identically to `==`.

> ⚠️ **Pitfall:** Pair with the `Integer` cache gotcha (Q48) — `Integer a=127,b=127; a==b` → `true`, but `200` → `false`.

### Q46. Full `equals()`/`hashCode()` contract
```
equals() contract:
1. Reflexive   : x.equals(x) → true
2. Symmetric   : x.equals(y) ⟺ y.equals(x)
3. Transitive  : x.equals(y) ∧ y.equals(z) → x.equals(z)
4. Consistent  : repeated calls return same result (no state change)
5. Null-safe   : x.equals(null) → false (never throws)

hashCode() contract:
6. Same object       → same hashCode (within one JVM run)
7. equals()==true     → hashCode() MUST also be equal (mandatory)
8. Same hashCode      → equals() may be true OR false (collisions fine)
```
```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Employee e)) return false; // null-safe + type check
    return id == e.id && Objects.equals(name, e.name);
}
@Override
public int hashCode() { return Objects.hash(id, name); }
```

> ⚠️ **Pitfall — HIGH VALUE:** Most candidates only know rule 7. Reciting **all 5 equals() rules + both hashCode() rules** unprompted is a strong depth signal. Also mention: **Java 14+ Records auto-generate both correctly**.

### Q26. Shallow copy vs deep copy (cloning)
- `Object.clone()` default = **shallow copy** — primitives by value, references shared with original.
- **Deep copy** recursively clones referenced mutable objects too.
- Shallow is safe for immutable fields only.

> ⚠️ **Pitfall:** Concrete failure: `order2.getItems().add(x)` also mutates `order1`'s list under shallow copy.

### Q27. [Follow-up] Implementing a deep copy
```java
@Override
public Order clone() {
    Order copy = (Order) super.clone();
    copy.items = new ArrayList<>(this.items);
    return copy;
}
```
- Alternatives: serialization-based deep copy, or (preferred by many) **copy constructors / static factory methods** instead of `clone()` entirely.

> ⚠️ **Pitfall:** Voice the informed opinion — Effective Java calls `Cloneable`/`clone()` "largely broken."

### Q28. What is `CloneNotSupportedException`?
- Checked exception from `Object.clone()` if the class doesn't implement `Cloneable` (a **marker interface** with no methods, checked at runtime by native `clone()`).

> ⚠️ **Pitfall:** This is exactly why `Cloneable` is considered an awkward interface — it doesn't declare `clone()` itself, just gates `Object.clone()`'s runtime behavior.

---

## 3. Strings, Memory & Security

### Q8. What is the String Pool?
- Special memory region (heap since Java 7, previously PermGen) caching unique **String literals**.
- Literals auto-interned at compile/class-load time; `new String(...)` bypasses the pool; `.intern()` manually pools.

> ⚠️ **Pitfall:** Classic gotcha — `"a"+"b" == "ab"` → often `true` (constant folding), `new String("ab") == "ab"` → `false`. Have both examples ready.

### Q13. Why is `char[]` preferred over `String` for passwords?
- Strings are **immutable + pooled** — may live in memory indefinitely, invisible to app control.
- `char[]` can be **explicitly zeroed** (`Arrays.fill(pw, '0')`) right after use.
- Strings show up more easily in heap dumps/logs/stack traces.

> ⚠️ **Pitfall — KEY POINT:** The real reason is **active zeroing capability**, not "strings are immutable" alone.

### Q53. Floating point pitfalls — NaN, Infinity, 0.1+0.2
```java
0.1 + 0.2 == 0.3;     // FALSE — 0.30000000000000004
Double.NaN == Double.NaN;  // FALSE! NaN never equals itself
Double.isNaN(nan);         // TRUE — the ONLY correct check
1.0 / 0.0;   // POSITIVE_INFINITY, no exception
10 / 0;      // ArithmeticException (int division DOES throw)
0.0 / 0.0;   // NaN
```
For money: use `BigDecimal` with the **String constructor**, never the double constructor:
```java
new BigDecimal("0.1").add(new BigDecimal("0.2")); // correct
new BigDecimal(0.1);                               // WRONG — imprecision baked in
```

> ⚠️ **Pitfall — HIGH VALUE:** `nan == nan` being `false` catches people off guard. `new BigDecimal(0.1)` "looking correct" while being wrong is a real production bug.

---

## 4. Casting & Type Conversion

### Q35. Casting — two broad categories
- **Primitive casting** (int/double/char etc.) vs **reference/object casting** (related class/interface types).
- Primitive casting can lose data; reference casting never touches the object's bytes.

> ⚠️ **Pitfall:** Don't conflate the two categories in your answer.

### Q36. Widening (implicit) cast
- `byte → short → int → long → float → double` (char folds into int side). Automatic, generally no data loss.

> ⚠️ **Pitfall:** `int→float` / `long→float/double` **can lose precision** even though "widening" — not always lossless.

### Q37. Narrowing (explicit) cast & overflow
- Requires explicit `(type)`. Overflow is **defined**: JLS specifies **modular truncation**, not clamping.
- `(byte)(1000d)` → `-24` (wraps, doesn't clamp to `Byte.MAX_VALUE`).

> ⚠️ **Pitfall:** Have the exact wrap-around number ready — reciting "might lose data" without a number is the weak answer.

### Q38. Reference (object) casting vs primitive casting
- Only changes **how you refer to** the object; heap object untouched.
- Only legal between inheritance-related types; unrelated classes → **compile-time error**.

### Q39. Upcasting
- Reference cast to a **supertype** — always safe, implicit, no `(Type)` needed.
- `Integer i=7; Number num=i; Object obj=i;` all implicit.

> ⚠️ **Pitfall:** After upcasting, you can only call methods on the **reference's static type**, even though the real object is still the subtype underneath.

### Q40. Downcasting
- Reference cast to a **subtype** — requires explicit cast; compiler can't verify at compile time.
- Wrong assertion → **`ClassCastException`** at runtime.
- Sibling trick: `B b=new B(); A a=b; C c=(C)a;` (B, C both extend A, unrelated) — **compiles**, throws at runtime.

> ⚠️ **Pitfall — MEMORIZE THIS:** The sibling-cast trick question — compiler only checks "plausible given declared type," not actual runtime correctness.

### Q51. `(byte)(128)` — walk-through
```java
byte b = (byte)(128); // -128
```
128 = `10000000` (9 bits needed) → truncated to 8 bits → MSB=1 → two's complement → `-128`.

> ⚠️ **Pitfall:** Follow-up: `(byte)(200)` → `200-256 = -56`. General rule: keep ±`2^N` until back in range.

### Q52. `(int)(-2.9)` — truncation vs rounding
```java
(int)(-2.9);  // -2, NOT -3
```
Truncation moves **toward zero**, never toward negative infinity (`Math.floor()` differs).

> ⚠️ **Pitfall — VERY COMMON MISTAKE:** Confusing truncation with flooring; only diverges for negative numbers.

### Q54. `Integer.MIN_VALUE` negation — overflow trap
```java
Math.abs(Integer.MIN_VALUE) == Integer.MIN_VALUE; // TRUE — still negative!
```
No positive counterpart exists for `-2147483648` in two's complement — negation **silently overflows back to itself**.

> ⚠️ **Pitfall:** Use `Math.absExact()` (Java 15+) to throw `ArithmeticException` instead of silently wrong result.

---

## 5. Wrapper Classes, Autoboxing & Boxing Performance

### Q41. Wrapper classes — why Java needs them
- Let primitives be treated as `Object` — needed for generics (`List<Integer>`), collections, reflection.
- **Autoboxing/unboxing** (Java 5+) makes conversion mostly invisible, but each boxing = new heap object (except cached range).

> ⚠️ **Pitfall:** `Long sum=0L; for(...) sum += i;` silently boxes/unboxes every iteration — use a primitive accumulator instead.

### Q47. How autoboxing actually works
```java
int → Integer : Integer.valueOf(int)   // compiler inserts this
Integer → int : integerObj.intValue()  // and this
```
No JVM magic — plain compiler-inserted method calls.

> ⚠️ **Pitfall:** It's `Integer.valueOf()`, **not** `new Integer()` — this is exactly what routes through the cache.

### Q48. Integer cache — exact rules
```
Integer   : -128 to 127 (default), tunable via -XX:AutoBoxCacheMax=N
Boolean   : TRUE/FALSE singletons, always
Byte      : -128 to 127 (ALL byte values — fixed)
Short     : -128 to 127 (fixed)
Long      : -128 to 127 (fixed, NOT configurable)
Character : 0 to 127 (fixed)
Double    : NO cache
Float     : NO cache
```

> ⚠️ **Pitfall — SCOPE LIMITATION:** Only `Integer.valueOf(int)` (autoboxing) uses the cache. `new Integer(5)` always creates new object. `Integer.parseInt("5")` returns a primitive, no boxing at all.

### Q49. Real memory overhead of boxing
```
int      : 4 bytes
Integer  : 16 bytes (12-byte header + 4-byte field) → 4x overhead
int[1000]     : ~4 KB
Integer[1000] : ~20 KB → 5x overhead
```

> ⚠️ **Pitfall:** Have exact byte numbers ready — quantified answer beats "boxing has overhead."

### Q50. Silent NPE trap — ternary with boxed types
```java
Integer x = null;
int y = x != null ? x : 0;  // NullPointerException!

Integer y = x != null ? x : 0; // FIX — keep result type as Integer
```
Ternary result type is determined from **both branches at compile time** — one branch `Integer`, other `int` → forces unboxing regardless of which branch runs.

> ⚠️ **Pitfall — GENUINELY SURPRISING:** The explicit null check makes it *look* safe; the bug is about static type determination, not runtime branch selection.

### Q55. Summing `List<Integer>` efficiently (avoiding boxing in streams)
```java
// BAD — boxes/unboxes every reduce step
int sum = list.stream().reduce(0, Integer::sum);

// GOOD — mapToInt converts once, upfront
int sum = list.stream().mapToInt(Integer::intValue).sum();
```

> ⚠️ **Pitfall:** Connects directly to Q49 — `mapToInt`/`mapToLong`/`mapToDouble` exist specifically to avoid per-step boxing cost.

---

## 6. Annotations

### Q15. How to create a custom annotation
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Loggable {
    String value() default "";
}
```
Consumed via reflection at runtime or annotation processors at compile time.

> ⚠️ **Pitfall:** An annotation alone does nothing — it's inert until something actively reads it.

### Q16. Purpose of `@Retention`
- `SOURCE` (compiler-discarded), `CLASS` (default, kept in `.class` but not runtime-loaded), `RUNTIME` (reflection-visible).

> ⚠️ **Pitfall — REAL BUG:** Forgetting `RUNTIME` retention → reflection-based lookup silently finds nothing, no error.

### Q17. What does `@Target` do?
- Restricts which element types an annotation applies to; enforced at **compile time**.

> ⚠️ **Pitfall:** Can accept multiple types in an array: `@Target({METHOD, FIELD})`.

### Q58. Repeatable annotations
```java
@Repeatable(Roles.class)
@interface Role  { String value(); }
@interface Roles { Role[] value(); }

@Role("admin")
@Role("user")   // without @Repeatable → COMPILE ERROR
class UserController { }
```
Java 8+ lets the compiler auto-wrap repeated annotations into the container.

> ⚠️ **Pitfall:** `getAnnotationsByType()` returns all instances transparently regardless of `@Repeatable` vs manual container syntax.

### Q59. Annotation processors vs reflection-based reading

| | Reflection-based reading | Annotation Processor |
|---|---|---|
| When it runs | Runtime, every app start | Compile time, once |
| Output | Behavior (proxies, wiring) | **Generated source code** |
| Runtime cost | Real (reflection overhead) | **Zero** |
| Used by | Spring (classic), Jackson | Lombok, MapStruct, Dagger2, AutoValue |

> ⚠️ **Pitfall — KEY INSIGHT:** Lombok rewrites the **AST during compilation** — zero runtime reflection footprint. This tension is why Spring AOT/GraalVM native image exist in Spring Boot 3.x.

---

## 7. Class Loading, Reflection & Encapsulation

### Q18. `Class.forName()` vs `ClassLoader.loadClass()`
- `Class.forName(name)` loads **and initializes** (runs static initializer) by default.
- `ClassLoader.loadClass(name)` only **loads**, no initialization until first use.
- `Class.forName(name, initialize, loader)` overload can mimic lazy behavior.

> ⚠️ **Pitfall:** JDBC driver registration relies specifically on `Class.forName()` triggering the static initializer that self-registers with `DriverManager`.

### Q19. Types of class loaders
- **Bootstrap** (core JDK, native code), **Platform/Extension**, **Application/System** (classpath, default for user code), **Custom** (app servers, plugins).
- **Parent-delegation model**: request delegates up to parent first.

> ⚠️ **Pitfall:** Explain *why* delegation exists — prevents a malicious/accidental `java.lang.String` on classpath from shadowing the real one.

### Q20. Can a class be loaded by two ClassLoaders?
- **Yes** — same FQN loaded by two unrelated loaders → JVM treats as **two distinct types**.
- Causes `ClassCastException` with **identical-looking class names on both sides**.

> ⚠️ **Pitfall — STRONG SENIOR SIGNAL:** Recognizing this exact symptom pattern instantly.

### Q44. Class loading — 3 phases in order
```
1. LOADING        — ClassLoader reads .class, creates Class object on heap
2. LINKING
   a. Verification — bytecode correctness check
   b. Preparation   — static fields set to DEFAULTS (0/null), not real values
   c. Resolution    — symbolic → direct references
3. INITIALIZATION — static initializers + assignments, TEXTUAL ORDER, ONCE, thread-safe
```

> ⚠️ **Pitfall:** Preparation-vs-Initialization is the detail most collapse into one step — defaults happen first, real values later.

### Q45. When does class initialization trigger?
```
- First instance creation (new)
- First static method call
- First static field READ/WRITE (except compile-time constants)
- When a subclass initializes (parent ALWAYS initializes first)
- Class.forName() (reflection)
```

> ⚠️ **Pitfall:** Compile-time constants (`static final int X = 5`) are **inlined** by the compiler — accessing them does NOT trigger the owning class's initialization.

### Q21. Using Reflection to break encapsulation
```java
Field f = obj.getClass().getDeclaredField("privateField");
f.setAccessible(true);
Object value = f.get(obj);
f.set(obj, newValue);
```
Used legitimately by DI frameworks, ORMs, test frameworks — but can violate invariants.

> ⚠️ **Pitfall:** Java 9+ module system can **block** this for non-`opens` packages — no longer an unconditional bypass.

### Q43. [Follow-up] Accessing a private constructor from outside
```java
Constructor<Singleton> constructor = Singleton.class.getDeclaredConstructor();
constructor.setAccessible(true);
Singleton instance = constructor.newInstance();
```
Same mechanism as Q21 — a private constructor is a **convention-level** safeguard, not absolute.

> ⚠️ **Pitfall:** Same Java 9+ module caveat applies.

### Q56. [Follow-up] Java 9+ module restriction on reflection & bypass
- Strong encapsulation: classes in a named module aren't reflectively accessible from outside **even with `setAccessible(true)`** → throws `InaccessibleObjectException`.
```java
// module-info.java
module com.example.myapp {
    opens com.example.internal to spring.core, com.fasterxml.jackson.databind;
}
// Emergency JVM flag:
// --add-opens java.base/java.lang=ALL-UNNAMED
```

> ⚠️ **Pitfall — WORTH MENTIONING UNPROMPTED:** This is a primary reason most Spring Boot apps don't actually adopt JPMS for application code — reflection-heavy frameworks rely on exactly this access.

### Q57. [Follow-up] Real risk of breaking encapsulation via reflection
```java
Field f = String.class.getDeclaredField("value"); // Java 8, pre-module restriction
f.setAccessible(true);
char[] val = (char[]) f.get(internedHelloString);
val[0] = 'X'; // MUTATES the shared, pooled "hello" String!
```
Mutating an **interned/pooled** String corrupts it for **every piece of code in the JVM** referencing that literal.

> ⚠️ **Pitfall — GENUINELY CATASTROPHIC, NOT THEORETICAL:** Have this ready when asked "why does immutability matter" beyond thread-safety.

### Q23. Troubleshooting `ClassNotFoundException`/`NoClassDefFoundError` post-deploy
- Check classpath/dependency mismatches, fat-jar/shaded-jar misconfigurations.
- `NoClassDefFoundError`: check if the class's **static initializer failed earlier** — search logs for an earlier `ExceptionInInitializerError`.
- Check classloader isolation (Q20) in app servers.
- Verify artifact contents: `jar tf app.jar | grep ClassName`.

> ⚠️ **Pitfall — MOST MISSED STEP:** Searching backward in logs for the original triggering static-initializer failure, instead of staring at the current stack trace.

---

## 8. Dynamic Proxies

### Q22. Dynamic proxies — what and how used
```java
MyInterface proxy = (MyInterface) Proxy.newProxyInstance(
    classLoader,
    new Class<?>[]{ MyInterface.class },
    (proxyObj, method, args) -> {
        System.out.println("before " + method.getName());
        Object result = method.invoke(realObject, args);
        System.out.println("after");
        return result;
    });
```
`java.lang.reflect.Proxy` generates a runtime class implementing interfaces, routing calls through an `InvocationHandler`. Powers Spring AOP, Mockito interface mocks.

> ⚠️ **Pitfall:** Only works for **interfaces** — proxying a concrete class needs CGLIB-style subclassing (what Spring falls back to for `@Transactional` without an interface).

### Q33. Role of `ClassLoader` in dynamic proxy creation
- `Proxy.newProxyInstance()` needs a `ClassLoader` because it **generates a new class at runtime** that must be defined into a classloader's namespace.
- Typically pass the target interface's own classloader.

> ⚠️ **Pitfall:** The question as usually phrased compares Proxy to itself (confused framing) — the real meaningful contrast is **JDK dynamic proxies (interface-based) vs. CGLIB/ByteBuddy (class subclassing)**, which is what Spring's `@Transactional`/`@Cacheable` decision hinges on.

---

## 9. I/O

### Q34. Role of `BufferedReader`; why preferred over `FileReader`
- `FileReader` reads char data directly — inefficient, near-per-char syscalls.
- `BufferedReader` wraps another `Reader`, buffers larger chunks, drastically reduces I/O ops. Adds `readLine()`.

> ⚠️ **Pitfall — ARCHITECTURAL SIGNAL:** Frame as the **Decorator pattern** applied to I/O — same pattern behind `BufferedOutputStream`/`BufferedInputStream` throughout `java.io`.

---

## 10. Marker Interfaces & Design Concepts

### Q12. Marker interfaces
- Interfaces with **no methods** — tag a class for metadata other code checks via `instanceof`. E.g. `Serializable`, `Cloneable`, `RandomAccess`.
- Since Java 5, annotations have largely superseded them for new APIs.

> ⚠️ **Pitfall:** Explain *why* — annotation = pure metadata; marker interface = permanently affects the type hierarchy (`instanceof`), sometimes deliberately desired (like `Serializable`).

### Q31. Circular constructor dependency between two classes
- Plain constructors can't resolve — genuine chicken-and-egg problem.
- Fixes: setter/field injection for one side, introduce an abstraction, or (senior answer) treat it as a **design smell** — extract a coordinating third class.
- Spring: constructor-injection cycles fail context startup by default; setter injection is a workaround, not the real fix.

> ⚠️ **Pitfall — SENIOR ANSWER:** The real fix is recognizing poor separation of concerns, not just reciting the wiring hack.

### Q32. "Ghost methods" — compiler optimization?
- Not a standard Java term — **say so honestly** rather than inventing a definition.
- Closest real concepts: **bridge methods** (generic type erasure / covariant returns) or **synthetic methods** (compiler-generated, e.g. nested-class private access).

> ⚠️ **Pitfall:** The right interview instinct — admit unfamiliarity, then pivot to the closest concept you do know.

---

## 11. Debugging & Production Troubleshooting (Experience-Based)

### Q14. JUnit fails as expected, but step-over debugging doesn't reproduce it
- Timing-sensitive/concurrency bugs — debugger changes thread interleaving.
- JIT compilation differences — debugger may force interpreted mode.
- Stale shared static state between test runs.
- Non-deterministic dependencies (system time, random seed, filesystem order).

> ⚠️ **Pitfall — LEAD WITH THESE:** Race conditions + JIT timing differences are the two most likely intended answers.

### Q24. Performance optimizations you've implemented
- Connection/thread pool tuning (HikariCP sized to real concurrency).
- Reducing N+1 in JPA/Hibernate (batch fetch, `@EntityGraph`, projection DTOs).
- Caching hot data (Redis/Caffeine, TTLs, cache-aside).
- JVM tuning (heap sizing, G1 vs ZGC, `-Xlog:gc`).
- Async/non-blocking I/O (`CompletableFuture` for parallel downstream calls).
- Profile before optimizing (async-profiler, JFR).

> ⚠️ **Pitfall:** This is experience-based — have **one concrete story with measurable before/after** (e.g. P99 latency X→Y), not just a generic list.

### Q25. Production server stopped logging & unresponsive — investigation steps
1. **Thread dump first** (`jstack <pid>` / `kill -3 <pid>`) — check deadlocks, all threads stuck on same resource.
2. Check GC logs for a long/continuous full GC ("stop-the-world").
3. Check OS resource exhaustion — disk full, fd limits, swapping.
4. Check if request queue/thread pool is saturated (no timeout on external calls).

> ⚠️ **Pitfall — ALWAYS FIRST STEP:** Thread dump before restarting — restarting destroys the evidence needed to fix the root cause.

---

*Source: Notion "Chapter 2 — Core Java" (59 questions). Original toggle order preserved via Q-numbers for cross-reference; grouped here by topic for revision efficiency.*
