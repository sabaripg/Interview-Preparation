# OOP Fundamentals — Abstract Classes, Interfaces, Polymorphism

> Abstract class vs interface (including the post-Java-8 default-method overlap and where the two still genuinely differ), the two kinds of polymorphism, and the precise overloading vs overriding resolution rules. Interview Q&A at the end.

## Abstract Class vs Interface — What's Actually Still Different

**Before Java 8**, the distinction was simple: abstract classes could have state and concrete method implementations; interfaces were pure contracts (only abstract method signatures, plus constants). **Since Java 8** (`default`/`static` interface methods) and **Java 9** (`private` interface methods), interfaces can carry real implementation code too — which is exactly why "interfaces can't have implementation" is now an outdated answer, and why the real distinction is subtler than it used to be.

```java
abstract class Vehicle {
    protected String registrationNumber; // abstract classes CAN hold instance state
    public Vehicle(String reg) { this.registrationNumber = reg; } // and constructors
    abstract void start(); // must be implemented by concrete subclasses
    void displayRegistration() { System.out.println("Reg: " + registrationNumber); } // concrete method, inherited as-is
}

interface Payable {
    double calculatePay(); // abstract -- implicitly public
    default double calculateBonus() { return calculatePay() * 0.1; } // default method -- CAN have a body
    static Payable zero() { return () -> 0.0; } // static factory method
    // NO instance fields with state allowed -- only public static final constants
}
```
**What genuinely still differs, precisely:**

```
| | Abstract Class | Interface |
|---|---|---|
| Instance state (non-constant fields) | Yes | No -- only `public static final` constants |
| Constructors | Yes | No |
| Multiple inheritance | No -- single inheritance only | Yes -- a class can implement many interfaces |
| Access modifiers on methods | Any (`public`/`protected`/`private`) | Implicitly `public` for abstract methods (`private` allowed since Java 9, but only for internal helper methods used by default methods) |
| "Is-a" vs "can-do" | Models a genuine type hierarchy | Models a capability/contract, often across unrelated types |
```
> ⚠️ **Pitfall — the outdated answer that signals stale knowledge:** "interfaces can't have method bodies" was true before Java 8 but is a dated statement today — an interviewer asking this in 2026 is checking whether you know about `default`/`static`/`private` interface methods, not testing the pre-Java-8 rule. The real, still-current distinction is about **state** (interfaces can't hold instance fields) and **multiple inheritance** (a class can implement unlimited interfaces but extend only one class) — lead with those, not "no method bodies."

## Why Multiple Inheritance of *Classes* Is Disallowed, But Multiple Interfaces Are Fine

**The Diamond Problem, precisely:** if a class could extend two classes that both define a field or concrete method with the same name, the compiler would have no unambiguous way to decide which one "wins" when accessed through the subclass — a genuine structural ambiguity, not just a style concern.

```java
interface A { default String greet() { return "Hello from A"; } }
interface B { default String greet() { return "Hello from B"; } }
class C implements A, B {
    // MUST override greet() explicitly -- the compiler refuses to guess which default wins
    @Override public String greet() { return A.super.greet(); } // explicit disambiguation syntax
}
```
Interfaces sidestep the *field*-collision half of the diamond problem entirely (no instance state to collide over), but a `default`-method name collision across two implemented interfaces still requires **explicit** disambiguation — the compiler forces you to override and choose, via the `InterfaceName.super.method()` syntax, rather than silently picking one.

## The Two Kinds of Polymorphism

**Compile-time (static) polymorphism — method overloading:** multiple methods with the same name, different parameter lists, resolved by the compiler based on the **declared/static types** of the arguments at the call site.
```java
void process(int x) { System.out.println("int version"); }
void process(String x) { System.out.println("String version"); }
process(5);       // "int version" -- resolved at COMPILE TIME
process("hello"); // "String version"
```
**Runtime (dynamic) polymorphism — method overriding:** a subclass provides its own implementation of a method already defined in its superclass/interface; which implementation actually runs is resolved by the **actual runtime type** of the object, not the reference's declared type.
```java
class Animal { String sound() { return "..."; } }
class Dog extends Animal { @Override String sound() { return "Woof"; } }
class Cat extends Animal { @Override String sound() { return "Meow"; } }

Animal a = new Dog(); // declared type Animal, runtime type Dog
System.out.println(a.sound()); // "Woof" -- resolved at RUNTIME via the object's actual type, NOT the reference's declared type
```
> ⚠️ **Pitfall — the precise mechanism behind runtime polymorphism, worth naming explicitly:** overriding works via the JVM's **virtual method table (vtable)** — every object carries a reference to its actual class's method table, and a call through an overridden method is dispatched via that table at runtime, not resolved statically at compile time the way overloading is. This is exactly why `a.sound()` above calls `Dog`'s version despite `a`'s *declared* type being `Animal` — the compiler only checks that `Animal` has a `sound()` method to call at all; *which* implementation actually runs is a runtime decision based on the vtable of the real object.

## Method Overloading vs Overriding — the Rules That Actually Get Checked

```java
class Parent {
    void show(int x) { System.out.println("Parent int: " + x); }
}
class Child extends Parent {
    @Override void show(int x) { System.out.println("Child int: " + x); } // OVERRIDE -- same signature
    void show(String x) { System.out.println("Child String: " + x); }     // OVERLOAD -- different signature, new method
}
```
**Overriding requires:** same method name, same parameter list (exact types, exact order), covariant or identical return type, and — for checked exceptions specifically — the overriding method cannot throw new or broader checked exceptions than the parent declared (see `core-java/Exception-Handling.md` for the full rule and why it follows directly from Liskov substitution). Access modifier can only stay the same or become **more** visible in the override, never less (`protected` → `public` is fine; `protected` → `private` is a compile error).

**Overloading requires:** same method name, but a **different** parameter list (different count, different types, or different order) — return type alone is **not** enough to distinguish an overload.
```java
int compute(int a) { return a; }
double compute(int a) { return a; } // COMPILE ERROR -- return type alone doesn't create a distinct overload
```
> ⚠️ **Pitfall — the classic overload-resolution trap with autoboxing and varargs:** when multiple overloads could match a given call, Java's resolution order is: (1) exact match with no boxing/unboxing and no varargs, (2) match allowing autoboxing/unboxing, (3) match allowing varargs — checked in that strict order, most-specific-first.
```java
void call(int x) { System.out.println("int"); }
void call(Integer x) { System.out.println("Integer"); }
void call(int... x) { System.out.println("varargs"); }
call(5); // "int" -- exact primitive match wins over both boxing AND varargs, even though all three technically apply
```
Getting this resolution order wrong (assuming autoboxing or varargs would be preferred over an exact match) is a common source of "why did it call *that* overload" confusion.

## Initialization Order Across Inheritance — The Classic Tricky Walkthrough

**The precise order, top to bottom, the very first time a subclass object is constructed:**
1. Parent's `static` fields + `static` initializer blocks (declaration order) — **only once**, the first time the parent class is loaded.
2. Child's `static` fields + `static` initializer blocks (declaration order) — also only once.
3. Parent's instance fields + instance initializer blocks (declaration order).
4. Parent's constructor body.
5. Child's instance fields + instance initializer blocks (declaration order).
6. Child's constructor body.

```java
class Parent {
    static { System.out.println("1. Parent static block"); }
    { System.out.println("3. Parent instance block"); }
    Parent() { System.out.println("4. Parent constructor"); }
}
class Child extends Parent {
    static { System.out.println("2. Child static block"); }
    { System.out.println("5. Child instance block"); }
    Child() { System.out.println("6. Child constructor"); }
}
new Child();
new Child(); // second instance
```
**Output:**
```
1. Parent static block
2. Child static block
3. Parent instance block
4. Parent constructor
5. Child instance block
6. Child constructor
3. Parent instance block
4. Parent constructor
5. Child instance block
6. Child constructor
```
> ⚠️ **Pitfall — the trap in the second `new Child()`:** static blocks run **exactly once per classloader**, regardless of how many instances are created — the second construction skips straight to steps 3–6. Relying on a static block for "setup that should happen once per instance" is a real, if now-obvious-in-hindsight, class of bug — static initialization is per-*class*, not per-*object*.

> ⚠️ **Pitfall — the genuinely dangerous version of this rule:** if a `Parent` constructor calls an overridable method that a `Child` override depends on one of `Child`'s own instance fields, that field is **not yet initialized** when `Parent`'s constructor runs (`Child`'s instance initializers/constructor haven't executed yet) — the override runs against the field's *default* value (`null`/`0`/`false`), not whatever the `Child` constructor was about to set it to. This is a classic, hard-to-spot NPE or silently-wrong-default bug, and the underlying rule ("never call overridable methods from a constructor") exists specifically because of it.

## Covariant Return Types

Since Java 5, an overriding method may return a **narrower** (subtype) return type than the method it overrides, as long as the narrower type is actually a subtype of the original declared return type.

```java
class Animal { Animal reproduce() { return new Animal(); } }
class Dog extends Animal {
    @Override Dog reproduce() { return new Dog(); } // covariant return -- Dog IS-A Animal, so this is a valid override
}
```
This is exactly the mechanism modern `Object.clone()` overrides rely on (returning the real subtype directly instead of `Object`, avoiding a manual downcast at every call site), and it's part of why records' auto-generated accessor methods and builder-style fluent APIs can chain cleanly across a type hierarchy.

> ⚠️ **Pitfall:** covariant returns only work in the *narrowing* direction (subtype), and only for reference types — returning a **wider** type than the overridden method declared is a plain compile error, since the override can't legally promise less than its supertype's contract guarantees to every existing caller.

## Composition Over Inheritance — the Fragile Base Class Problem

**The problem:** deep inheritance couples a subclass to the *exact implementation details* of its superclass, not merely its public contract. A change to a base class's method **body** (not its signature) that looks like safe, private refactoring can silently break every subclass that unknowingly depended on the old internal behavior — because subclasses aren't just external callers, their own code runs interleaved with the superclass's.

**The textbook cautionary example, straight from the JDK itself:** `java.util.Properties extends Hashtable<Object,Object>` — meaning a `Properties` instance technically inherits `put(Object, Object)` and can have non-`String` keys/values shoved into it via the inherited `Hashtable` API, silently violating `Properties`' own documented "keys and values are Strings" contract. This is one of the most commonly cited "should have been composition" regrets in the standard library.

**The fix — composition:** instead of `class Stack<T> extends ArrayList<T>` (which leaks every `ArrayList` method, including ones that violate stack discipline entirely, like inserting at an arbitrary index), wrap a `List<T>` as a **private field** and expose only the methods a stack actually needs (`push`, `pop`, `peek`) — the wrapped list's full API never leaks to callers.

> ⚠️ **Pitfall — this is not a blanket "never use inheritance" rule:** *Effective Java*'s "favor composition over inheritance" guidance specifically targets inheriting from a class **you don't control**, or one that isn't a genuine, tightly-scoped specialization evolving in lockstep with its parent. Inheriting within your own codebase, between classes you own and version together, carries a much smaller version of this risk — the rule is about unpredictable coupling to code outside your control, not inheritance as a concept.

---

## Interview Q&A

**Q: Since Java 8 interfaces can have method bodies via `default` methods — so what's actually still different from an abstract class?**
Interfaces still can't hold instance state (only `public static final` constants) and can't have constructors. The bigger structural difference is multiple inheritance: a class can implement any number of interfaces but extend only one class — interfaces model a capability/contract that can apply across unrelated type hierarchies, abstract classes model a genuine "is-a" hierarchy with shared state.

**Q: If a class implements two interfaces that both define the same default method, what happens?**
Compile error unless the class explicitly overrides the method and disambiguates which one to use (or provides its own), via `InterfaceName.super.method()` syntax. The compiler refuses to silently pick one — this is the interface-level echo of the classic multiple-inheritance diamond problem, though narrower since interfaces hold no state to collide over.

**Q: Overloading vs overriding — how is each actually resolved, compile-time or runtime?**
Overloading (same name, different parameter list) is resolved at compile time based on the declared/static types of the arguments. Overriding (subclass redefining a superclass/interface method with an identical signature) is resolved at runtime via the object's actual type, dispatched through the JVM's virtual method table — this is why a reference declared as the supertype still calls the subtype's overridden implementation.

**Q: Can two methods differ only by return type and count as valid overloads?**
No — return type alone doesn't distinguish an overload; the parameter list (count, types, or order) must differ. A method differing only in return type with an identical parameter list is a compile error, not a valid overload.

**Q: Given overloads for `int`, `Integer`, and `int...` (varargs) all matching a call like `call(5)`, which one runs?**
The exact primitive match (`int`) — Java's overload resolution checks exact matches (no boxing, no varargs) first, then autoboxing/unboxing matches, then varargs matches, in that strict most-specific-first order, regardless of declaration order in the source file.

**Q: Walk through the exact initialization order when a subclass object is constructed for the first time.**
Parent static fields/blocks → child static fields/blocks (both only once, ever, per classloader) → parent instance fields/blocks → parent constructor → child instance fields/blocks → child constructor. On a *second* instantiation, both static steps are skipped entirely — they already ran.

**Q: Why is calling an overridable method from a constructor dangerous?**
If a superclass constructor calls a method the subclass overrides, and that override depends on a subclass instance field, the field is still at its default value (`null`/`0`/`false`) — the subclass's own field initializers and constructor haven't run yet at that point in the sequence. This produces a real, hard-to-diagnose NPE or silently-wrong-default bug.

**Q: What are covariant return types, and where does the JDK itself rely on them?**
An overriding method may return a subtype of the return type its superclass/interface declared. Modern `Object.clone()` overrides use this to return the real concrete type directly instead of `Object`, avoiding a manual cast at every call site.

**Q: Why does *Effective Java* recommend composition over inheritance, and is it an absolute rule?**
Because subclassing couples you to a superclass's exact implementation details, not just its public contract — an internal behavior change in the superclass can silently break subclasses that depended on the old behavior, even without any signature change (`Properties extends Hashtable` is the JDK's own cited regret). It's not absolute: the risk is much lower when both classes are in your own codebase and evolve together deliberately, versus inheriting from external code you don't control.
