# Reflection

> What Reflection is, how to get a `Class` object three ways, inspecting/invoking methods, fields, and constructors — including private ones — plus the senior-level performance and security angles the basic walkthrough usually skips. Interview Q&A at the end.

## What Reflection Is

**What it does:** Reflection lets a running program inspect and manipulate its own classes, methods, fields, and constructors at runtime — including ones it couldn't reference directly at compile time. Concretely, it can tell you what methods a class has, what a method's return type and modifiers are, what interfaces a class implements, and it can read or write field values (even `private` ones) and invoke methods dynamically by name.

**Why it matters beyond "cool trick":** almost every framework you use daily runs on reflection under the hood — Spring's dependency injection (`@Autowired` resolves and injects fields via reflection), Jackson/Gson (serializing objects by reflecting over their fields), JUnit (finding and invoking `@Test`-annotated methods), and Hibernate (populating entity fields without calling your constructors or setters). Understanding reflection well is understanding how these frameworks actually work, not just how to use them.

## The `Class` Object — the Foundation

**What it is:** every loaded class has exactly one corresponding `java.lang.Class` instance, created by the JVM at class-loading time. This object is the entry point for all reflection — it carries the class's metadata (methods, fields, constructors, modifiers, superclass, interfaces).

**Three ways to get it:**

```java
// 1. Class.forName() — loads the class dynamically by fully-qualified name, if not already loaded
Class<?> c1 = Class.forName("com.example.Employee");

// 2. .class literal — resolved at compile time, no class loading triggered beyond what's already needed
Class<?> c2 = Employee.class;

// 3. getClass() — from an existing instance, always returns the RUNTIME type, not the reference type
Employee emp = new Manager(); // reference type Employee, runtime type Manager
Class<?> c3 = emp.getClass(); // returns Manager, not Employee
```

> ⚠️ **Pitfall:** these three are not interchangeable in every context. `Class.forName()` actively triggers class loading (running static initializers) if the class isn't loaded yet — this is exactly how JDBC drivers used to self-register before `ServiceLoader` became standard. `.class` and `getClass()` don't force loading beyond what's already happened. Also, `getClass()` returning the *runtime* type (not the declared reference type) is the detail interviewers check — it's why polymorphic dispatch and reflection-based type checks can disagree with what the code "looks like" it's doing.

## Accessing Methods, Fields, and Constructors — Two Method Families

**The core distinction:** every reflective lookup comes in a `get*()` and a `getDeclared*()` variant, and confusing them is the single most common reflection mistake.

| | `getMethods()` / `getFields()` / `getConstructors()` | `getDeclaredMethods()` / `getDeclaredFields()` / `getDeclaredConstructors()` |
|---|---|---|
| Visibility | **public only** | **all** — public, protected, package-private, private |
| Inheritance | Includes inherited public members from superclasses | **Only** members declared directly in this class — nothing inherited |

```java
class Base {
    public void publicBaseMethod() {}
    private void privateBaseMethod() {}
}
class Employee extends Base {
    public void publicMethod() {}
    private void privateMethod() {}
}

Method[] pub = Employee.class.getMethods();          // publicMethod + publicBaseMethod (inherited) + Object's methods
Method[] all = Employee.class.getDeclaredMethods();   // publicMethod + privateMethod ONLY — no inherited methods at all, public or not
```
> ⚠️ **Pitfall:** `getDeclaredMethods()` does **not** give you "everything including inherited" — it's the opposite axis from `getMethods()`. To truly enumerate every member across a hierarchy (public and private, including inherited), you have to walk up the superclass chain yourself, calling `getDeclaredMethods()` at each level via `getSuperclass()`.

## Invoking Methods Reflectively

```java
import java.lang.reflect.Method;

public class Employee {
    private String name = "Sabari";
    public String greet(String prefix) { return prefix + ", " + name; }
    public static void main(String[] args) throws Exception {
        Employee emp = new Employee();
        Method greetMethod = Employee.class.getMethod("greet", String.class); // public, so getMethod() works
        Object result = greetMethod.invoke(emp, "Hello"); // (target instance, args...)
        System.out.println(result);
    }
}
```
**Output:**
```
Hello, Sabari
```
`invoke(target, args...)` is the universal call mechanism — the target instance is `null` for `static` methods. The return type is always `Object`, so callers must cast.

## Accessing Private Fields — `setAccessible(true)`

**What it does:** by default, the JVM's normal access-control checks (`private`/`protected`/package-private) still apply during reflective access — attempting to read/write a private field without permission throws `IllegalAccessException`. `setAccessible(true)` tells the JVM to bypass that check for this specific `Field`/`Method`/`Constructor` object.

```java
import java.lang.reflect.Field;

class Account {
    private double balance = 500.0;
}

public class PrivateFieldDemo {
    public static void main(String[] args) throws Exception {
        Account acc = new Account();
        Field balanceField = Account.class.getDeclaredField("balance"); // getDeclaredField -- private, so getField() would throw NoSuchFieldException
        // balanceField.get(acc); // would throw IllegalAccessException here without the next line
        balanceField.setAccessible(true); // bypasses the private access check
        System.out.println("Before: " + balanceField.get(acc));
        balanceField.set(acc, 750.0); // mutating a private field from outside the class entirely
        System.out.println("After: " + balanceField.get(acc));
    }
}
```
**Output:**
```
Before: 500.0
After: 750.0
```
> ⚠️ **Pitfall — this is precisely the "encapsulation isn't a security boundary" gotcha:** `private` in Java stops *accidental* misuse at compile time; it does not stop reflection at runtime. Frameworks rely on exactly this (Hibernate setting entity fields directly, Mockito injecting mocks into `@InjectMocks` fields) — but it also means a determined caller can always reach into any object's private state unless a `SecurityManager`/module boundary (Java 9+ strong encapsulation, `--illegal-access`) explicitly blocks it. As of Java 17+, the JDK's own internals are far more locked down against this than ordinary application code is.

## Reflective Constructors

```java
import java.lang.reflect.Constructor;

class Point {
    private int x, y;
    public Point(int x, int y) { this.x = x; this.y = y; }
    public String toString() { return "(" + x + "," + y + ")"; }
}

Constructor<Point> ctor = Point.class.getConstructor(int.class, int.class);
Point p = ctor.newInstance(3, 4); // equivalent to new Point(3, 4), but resolved at runtime
System.out.println(p);
```
**Output:**
```
(3,4)
```
This is exactly how Spring instantiates beans and Jackson instantiates deserialized objects when a no-arg constructor isn't available — `getDeclaredConstructor()` + `setAccessible(true)` + `newInstance()` is the standard pattern for frameworks needing to construct objects whose constructors aren't public.

## The Real Costs of Reflection — Why It's Not Free

**Performance:** every reflective call (`invoke()`, `get()`, `set()`) is dramatically slower than direct code — commonly cited at 10-100x overhead versus a direct method call or field access, because the JVM can't inline or optimize through the reflective indirection the way it can with normal bytecode. `Method.setAccessible(true)` also disables some JIT-level optimizations for that member. This is why performance-sensitive frameworks (modern Jackson, many DI containers) generate bytecode at startup (via ASM/ByteBuddy) to create direct accessor classes once, rather than paying the reflection cost on every single field access at runtime.

**Security and modularity:** since Java 9's module system, `setAccessible(true)` can be blocked entirely for JDK-internal classes unless the target module explicitly `opens` that package — this is why some libraries that worked fine on Java 8 broke or needed `--add-opens` flags on Java 9+.

> ⚠️ **Pitfall:** "reflection is just a bit slower" understates it for hot-path code — reflectively invoking a method inside a tight loop processing millions of records is a real, measurable production performance issue, not a theoretical one. Cache the `Method`/`Field` object (looked up once) rather than re-resolving it on every call, and consider whether reflection is even necessary versus a direct call, an interface, or a generated accessor.

## `MethodHandle` — the Modern, Faster Alternative to Reflection

**What it is:** `java.lang.invoke.MethodHandle` (Java 7+) is a lower-level, JVM-native mechanism for method invocation that the JIT can inline and optimize far more aggressively than classic `Method.invoke()` — `MethodHandle`s are, in fact, exactly what `invokedynamic` (and therefore lambda expressions — see `Java8-Features/Part2-Lambdas-Functional-Interfaces.md`) compile down to underneath.

```java
MethodHandles.Lookup lookup = MethodHandles.lookup();
MethodHandle greet = lookup.findVirtual(Employee.class, "greet", MethodType.methodType(String.class, String.class));
String result = (String) greet.invoke(new Employee(), "Hello");
```
**Why it's faster:** `Method.invoke()` goes through a generic, boxing-heavy call path that historically carried security-manager checks on every invocation (partially JIT-mitigated in modern JVMs, but still real overhead). A resolved `MethodHandle`, once warmed up by the JIT, behaves much closer to a direct method call, because the JVM has dedicated bytecode-level support for handle invocation that it never had for reflective `Method.invoke()`.

> ⚠️ **Pitfall — knowing it exists is the signal, not necessarily using it:** `MethodHandle`'s API is notoriously low-level and unfriendly compared to reflection (explicit `MethodType` construction, no simple `getDeclaredMethods()`-style enumeration) — it's the tool library and framework authors reach for once performance is *genuinely proven* to matter (modern JSON libraries, some DI containers, the `invokedynamic` machinery itself), not a drop-in replacement for everyday reflective code in application logic. For most application code, plain reflection (or better, avoiding the need for either) remains the right default — but recognizing `MethodHandle` by name, and *why* it's faster, is exactly the kind of ecosystem-currency signal a 10-YOE interview probes for.

---

## Interview Q&A

**Q: What is reflection, and where does the JVM actually use it besides "my own code calling it"?**
Reflection is runtime inspection/manipulation of classes, methods, fields, and constructors. Nearly every framework (Spring, Hibernate, Jackson, JUnit, Mockito) is built on it — dependency injection, ORM field population, JSON (de)serialization, and test discovery all work by reflecting over your classes rather than you writing that wiring by hand.

**Q: `Class.forName()` vs `.class` vs `getClass()` — what's the actual difference?**
Covered above. `Class.forName()` can trigger class loading/static-initializer execution if not already loaded; `.class` is compile-time and doesn't force loading; `getClass()` returns the object's *runtime* type, which can differ from its declared reference type under polymorphism.

**Q: `getMethods()` vs `getDeclaredMethods()` — what's the precise difference?**
Covered above. It's a two-axis distinction: visibility (public-only vs all) crossed with inheritance (includes inherited public members vs. only this class's own declarations) — `getDeclaredMethods()` is not simply "the private version of getMethods()."

**Q: Why does accessing a private field via reflection need `setAccessible(true)`, and what does that actually bypass?**
It bypasses the JVM's normal Java-language access-control check for that specific reflective object — not a security sandbox, just the same `private`/`protected` enforcement the compiler normally does. This is why `private` fields are not a hard security boundary against a determined caller, only a compile-time discipline tool — module-system encapsulation (Java 9+) is the actual hard boundary for JDK internals.

**Q: What's the real performance cost of reflection, and how do high-performance frameworks avoid paying it repeatedly?**
Reflective calls run materially slower than direct calls (often cited 10-100x) because the JIT can't optimize through the indirection as well. Frameworks amortize this by caching resolved `Method`/`Field` objects instead of re-resolving by name on every call, or by generating real bytecode accessors once at startup (ASM/ByteBuddy) instead of using raw reflection on every access.

**Q: What is `MethodHandle`, and why is it faster than classic reflection?**
A lower-level, JVM-native invocation mechanism (Java 7+) that the JIT can inline and optimize much more aggressively than `Method.invoke()` — it's the same underlying machinery `invokedynamic` and lambda expressions compile to. Once resolved and JIT-warmed, a `MethodHandle` call performs close to a direct method call, unlike reflective `Method.invoke()`, which pays a generic, boxing-heavy overhead on every call.
