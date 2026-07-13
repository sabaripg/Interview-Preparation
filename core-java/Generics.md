# Generics

> Generic classes/methods, bounded types, wildcards and the PECS principle, type erasure and everything it breaks (arrays, overloading, instanceof), heap pollution, and the restrictions erasure imposes. Interview Q&A at the end.

## Why Generics Exist

**Before Java 5**, collections stored `Object` — no compile-time type safety, and every read needed an explicit cast:
```java
ArrayList list = new ArrayList();
list.add("Java");
list.add(10); // compiles — the list has no idea what it's "supposed" to hold
String s = (String) list.get(1); // compiles, throws ClassCastException at RUNTIME
```
Generics move that failure from a runtime `ClassCastException` (discovered by a user, in production) to a compile-time error (discovered by the developer, immediately):
```java
List<String> list = new ArrayList<>();
list.add("Java");
list.add(10); // compile-time error — caught before the code ever runs
```

## Generic Classes and Generic Methods — Different Scopes

```java
// Generic CLASS — type parameter belongs to the whole class, fixed at instantiation
class Box<T> {
    private T value;
    void set(T value) { this.value = value; }
    T get() { return value; }
}
Box<String> box = new Box<>(); // T is bound to String for THIS box, forever

// Generic METHOD — type parameter belongs only to this method, inferred per call
public <T> void print(T value) { System.out.println(value); }
print("Java"); // T = String for this call
print(10);     // T = Integer for this call, independently
```
> ⚠️ **Pitfall:** a generic class's type parameter is fixed once at object creation; a generic method's type parameter is resolved fresh on every individual call. Mixing these up leads to confusion about why a "generic" class instance can't suddenly hold a different type on a later call — it was never meant to.

## Bounded Types

```java
class Calculator<T extends Number> {  // T must be Number or a subtype
    private T num;
    double asDouble() { return num.doubleValue(); } // legal -- Number guarantees doubleValue()
}
// Calculator<String> calc; // compile error -- String is not a Number
```
**Multiple bounds:** `<T extends Number & Comparable<T>>` — at most one *class* bound, and it must come first; any number of *interface* bounds after it.

```java
class RankedItem<T extends Number & Comparable<T>> {
    T value;
    boolean isBetterThan(T other) { return value.compareTo(other) > 0; } // needs Comparable
    double magnitude() { return value.doubleValue(); }                   // needs Number
}
```

## Wildcards and the PECS Principle

**Three wildcard forms**, each answering "what can I do with this collection of unknown-but-related type":

```java
List<?> anyList;                  // unbounded -- read as Object only, effectively read-only (only null can be added)
List<? extends Number> readable;  // upper-bounded -- accepts List<Integer>, List<Double>... can READ as Number, can't add
List<? super Integer> writable;   // lower-bounded -- accepts List<Integer>, List<Number>, List<Object>... can ADD Integer, reads only as Object
```

**PECS — Producer Extends, Consumer Super**, the mnemonic for choosing between them:

```java
// Producer of T (you only READ values out) -> use ? extends T
void printAll(List<? extends Number> numbers) {
    for (Number n : numbers) System.out.println(n); // safe: everything in there IS-A Number
}

// Consumer of T (you only WRITE values in) -> use ? super T
void addIntegers(List<? super Integer> list) {
    list.add(1);
    list.add(2); // safe: whatever the real list type is, it accepts Integer
}
```

**Why `List<? extends Number>` refuses `.add()`:**
```java
List<? extends Number> list = new ArrayList<Integer>();
list.add(10); // compile error
```
The compiler only knows "some subtype of Number" — it could actually be a `List<Double>` behind that reference. Allowing `add(10)` (an `Integer`) into what might really be a `List<Double>` would silently corrupt it. Only `add(null)` is ever allowed, since `null` is valid for every reference type.

**Why reading from `List<? super Integer>` only gives you `Object`:**
```java
List<? super Integer> list = new ArrayList<Number>();
Integer i = list.get(0); // compile error -- the compiler can't promise the real type IS Integer, only that it's SOME supertype
Object o = list.get(0);  // fine -- Object is the only guaranteed common ground
```

> ⚠️ **Pitfall:** PECS isn't an arbitrary mnemonic to memorize — it's a direct consequence of what the compiler can actually *prove* about an unknown related type. If you can't articulate *why* `extends` blocks writes and `super` blocks precise reads, you have the rule memorized but not understood — and that's exactly the follow-up senior interviewers ask.

## Type Erasure — the Foundational Mechanism Behind Every Generics Quirk

**What it is:** Java generics are a **compile-time-only** feature. The compiler enforces type safety at compile time, then erases the generic type information, replacing type parameters with their bound (or `Object` if unbounded) in the compiled bytecode.

```java
List<String> list = new ArrayList<>();
// compiles down to, effectively:
List list = new ArrayList(); // raw type at the bytecode level -- List<String> and List<Integer> are IDENTICAL after compilation
```

**Every one of the following restrictions is a direct consequence of erasure — not an arbitrary language rule:**

```java
// 1. Can't create a generic array -- there's no runtime type to make the array of
T[] arr = new T[10]; // compile error

// 2. Can't check instanceof against a parameterized type -- the parameter doesn't exist at runtime
if (list instanceof List<String>) { } // compile error
if (list instanceof List<?>) { }      // fine -- unbounded wildcard has nothing to check

// 3. Can't overload methods that only differ by generic parameter -- they have the SAME erased signature
void method(List<String> list) { }
void method(List<Integer> list) { } // compile error -- both erase to method(List)

// 4. Can't instantiate a type parameter directly
T obj = new T(); // compile error -- erasure means the JVM has no idea what class T actually is at runtime

// 5. Can't have a static field of the type parameter
class Box<T> { static T value; } // compile error -- static belongs to the class, but T differs per-instance-type conceptually

// 6. Can't catch a generic type as an exception
catch (T e) { } // compile error
```
> ⚠️ **Pitfall:** every one of these six restrictions is frequently tested as an isolated "gotcha," but they all reduce to the *same* root cause: by the time the JVM runs the code, `T` simply isn't there anymore. Leading with "because of type erasure" for any of these, rather than treating each as an unrelated arbitrary rule, is what demonstrates real understanding versus memorized trivia.

## Heap Pollution — What Erasure Actually Puts at Risk

**What it is:** a situation where a variable of a parameterized type ends up referring to an object that isn't actually an instance of that parameterization — usually via mixing raw types with generics.

```java
List<String> list = new ArrayList<>();
List raw = list;      // raw type reference to the same object -- legal, but dangerous
raw.add(10);           // compiles (raw types skip generic checks entirely) -- silently corrupts 'list'

String s = list.get(0); // ClassCastException at RUNTIME -- the Integer is really there, disguised as String
```
This is exactly the failure mode generics were introduced to prevent — and it can only resurface when raw types are deliberately or accidentally mixed back in with parameterized code (a common source: old pre-generics libraries, or careless casting).

## Bridge Methods — the Compiler's Patch for Erasure vs. Polymorphism

```java
class Parent<T> { T get() { return null; } }
class Child extends Parent<String> { @Override String get() { return "hi"; } }
```
After erasure, `Parent.get()` has signature `Object get()`, but `Child.get()` has signature `String get()` — these don't match at the bytecode level, which would break polymorphic dispatch (`Parent p = new Child(); p.get();` needs to call `Child`'s override). The compiler silently generates a synthetic **bridge method** — `Object get()` in `Child` that just calls the real `String get()` and returns its result — invisible in source code, purely there to keep override resolution working correctly despite erasure.

## Why Arrays Are Covariant but Generics Are Not

```java
Object[] arr = new String[10]; // legal -- arrays are covariant
arr[0] = 10;                    // compiles, throws ArrayStoreException at RUNTIME -- array remembers its real component type

List<String> list = new ArrayList<>();
List<Object> objList = list; // compile error -- generics are invariant, deliberately
```
Arrays carry their component type at runtime and check it on every store (allowing this unsafe-looking assignment to compile, but failing loudly later). Generics carry no such runtime information (erasure), so the language closes the hole entirely at compile time instead — generic type parameters are invariant unless a wildcard explicitly says otherwise (which is the entire reason wildcards with PECS exist).

## The Diamond Operator (Java 7+)

```java
List<String> list = new ArrayList<String>(); // pre-Java 7, redundant
List<String> list = new ArrayList<>();       // Java 7+, type inferred from the left side
```
Pure verbosity reduction — no behavior change, no erasure implications.

---

## Interview Q&A

**Q: What problem do generics solve, concretely?**
They move type-mismatch failures from a runtime `ClassCastException` (discovered in production, by a user) to a compile-time error (discovered immediately, by the developer) — plus eliminating the need for explicit casts on every read.

**Q: State the PECS principle and explain *why* it holds, not just what it says.**
Producer (read-only source) → `? extends T`; Consumer (write-only target) → `? super T`. It holds because the compiler can only guarantee what's *actually knowable* about an unknown related type: with `extends`, it knows every element IS-A T (safe to read as T, unsafe to add since the concrete subtype is unknown); with `super`, it knows the list accepts T (safe to add T), but can only guarantee reads are Object (the real type could be any supertype of T).

**Q: What is type erasure, and name three concrete restrictions it causes.**
Generic type parameters exist only at compile time and are erased (replaced by their bound or `Object`) in compiled bytecode. This is why you can't create a generic array, can't `instanceof` against a parameterized type, can't overload methods differing only by type parameter, can't `new T()`, can't have a `static` field of type `T`, and can't `catch (T e)`.

**Q: What is heap pollution, and how does it actually happen given that generics are supposed to prevent exactly this?**
It happens when a raw-type reference to the same underlying object bypasses generic type checking (e.g., `List raw = someGenericList;`), letting an incompatible element get added without a compile error — the resulting `ClassCastException` surfaces later, on read, exactly like pre-generics code. It's a deliberate escape hatch (for legacy interop) that reintroduces the exact risk generics exist to close.

**Q: Why are arrays covariant while generics are invariant — isn't that inconsistent?**
Arrays carry runtime type information and enforce it on every store, throwing `ArrayStoreException` if violated — so allowing the covariant assignment to compile is "safe" in the sense that a violation is caught immediately, just at runtime instead of compile time. Generics have no such runtime information after erasure, so the language closes the equivalent hole at compile time instead by making generics invariant by default — wildcards are the deliberate, explicit escape hatch when covariance/contravariance is actually needed.
