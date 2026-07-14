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
